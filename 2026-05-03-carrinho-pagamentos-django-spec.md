# Spec: Carrinho de Compras e Sistema de Pagamentos em Django

> Documento genérico de referência. Descreve como implementar um sistema completo de
> carrinho, checkout e pagamentos em qualquer projeto Django. Não é específico para
> nenhum negócio. Uma IA pode ler este documento e implementar o sistema do zero.

---

## 1. Visão Geral da Arquitetura

O sistema é dividido em dois apps Django:

| App | Responsabilidade |
|-----|-----------------|
| `commerce` | Carrinho, pedido, itens do pedido, comissões, métricas |
| `payments` | Métodos de pagamento, transações, webhooks, confirmação |

Um terceiro app (`notifications`) cuida do envio de e-mails transacionais com suporte a
templates editáveis via admin e idempotência por chave única.

### Fluxo de alto nível

```
[usuário] → adiciona item ao carrinho
         → cart_detail (GET /carrinho/)
         → checkout POST → create_order_from_cart()
               ↓
         Order (pending) + OrderItems + PaymentTransaction (pending)
               ↓
         [usuário paga manualmente: Pix / transferência]
               ↓
         [admin confirma] → confirm_payment()
               ↓
         Order (paid) + grant_order_access() + e-mail de acesso liberado
```

---

## 2. Modelos

### 2.1 `commerce/models.py`

#### Cart

Representa o carrinho ativo de um usuário ou de uma sessão anônima.

```python
from decimal import Decimal
from django.conf import settings
from django.db import models
from django.utils.translation import gettext_lazy as _

class Cart(models.Model):
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
        related_name="carts", null=True, blank=True,
    )
    session_key = models.CharField(max_length=80, blank=True)
    updated_at  = models.DateTimeField(auto_now=True)
    created_at  = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ("-updated_at",)

    def __str__(self):
        return self.user.email if self.user_id else f"Cart {self.pk}"

    @property
    def total(self) -> Decimal:
        return sum((i.line_total for i in self.items.all()), Decimal("0.00"))
```

**Regras:**
- `user` e `session_key` são mutuamente exclusivos na prática (mas não com constraint de banco
  — a lógica de views garante isso).
- Um usuário pode ter apenas um carrinho ativo. Carrinhos anônimos são identificados por
  `session_key`.
- Quando um usuário anônimo faz login, o carrinho de sessão é mesclado ao carrinho do usuário.

---

#### CartItem

Cada linha do carrinho. Suporta dois tipos de item: produto avulso **ou** plano de assinatura.

```python
class CartItem(models.Model):
    cart              = models.ForeignKey(Cart, on_delete=models.CASCADE, related_name="items")
    product           = models.ForeignKey("catalog.Product", on_delete=models.CASCADE,
                                          related_name="cart_items", null=True, blank=True)
    subscription_plan = models.ForeignKey("subscriptions.SubscriptionPlan", on_delete=models.CASCADE,
                                          related_name="cart_items", null=True, blank=True)
    quantity   = models.PositiveIntegerField(default=1)
    unit_price = models.DecimalField(max_digits=10, decimal_places=2, default=0)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=["cart", "product"],
                condition=models.Q(product__isnull=False),
                name="unique_cart_product",
            ),
            models.UniqueConstraint(
                fields=["cart", "subscription_plan"],
                condition=models.Q(subscription_plan__isnull=False),
                name="unique_cart_subscription_plan",
            ),
        ]

    def clean(self):
        has_product = bool(self.product_id)
        has_plan    = bool(self.subscription_plan_id)
        if has_product == has_plan:
            raise ValidationError("CartItem must have exactly one: product or subscription_plan.")

    @property
    def line_total(self) -> Decimal:
        return self.unit_price * self.quantity

    @property
    def display_title(self) -> str:
        if self.subscription_plan_id:
            return self.subscription_plan.name
        return self.product.title
```

**Regras:**
- O preço (`unit_price`) é **capturado no momento da adição**, não calculado dinamicamente.
  Isso evita que mudanças no catálogo afetem carrinhos ativos.
- Constraints `UniqueConstraint` com `condition` evitam duplicatas no banco sem impedir NULL.

---

#### Order

Pedido imutável criado a partir do carrinho no momento do checkout.

```python
class Order(models.Model):
    STATUS_CHOICES = [
        ("draft",    _("Draft")),
        ("pending",  _("Pending")),
        ("paid",     _("Paid")),
        ("failed",   _("Failed")),
        ("refunded", _("Refunded")),
    ]
    PAYMENT_METHOD_CHOICES = [
        ("pix",           "Pix"),
        ("card",          _("Credit Card")),
        ("bank_transfer", _("Bank Transfer")),
    ]
    ACQUISITION_SOURCE_CHOICES = [
        ("web",    _("Web Portal")),
        ("mobile", _("Mobile App")),
    ]

    user               = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                                           related_name="orders")
    status             = models.CharField(max_length=20, choices=STATUS_CHOICES, default="pending")
    total_amount       = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    payment_method     = models.CharField(max_length=20, choices=PAYMENT_METHOD_CHOICES, blank=True)
    acquisition_source = models.CharField(max_length=20, choices=ACQUISITION_SOURCE_CHOICES, default="web")
    coupon_code        = models.CharField(max_length=40, blank=True)
    paid_at            = models.DateTimeField(null=True, blank=True)
    expires_at         = models.DateTimeField(null=True, blank=True)
    checkout_snapshot  = models.JSONField(default=dict, blank=True)
    notes              = models.TextField(blank=True)
    created_at         = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ("-created_at",)
```

**Campos importantes:**

| Campo | Propósito |
|-------|-----------|
| `checkout_snapshot` | JSON com os itens e total no momento do checkout — preserva o estado mesmo que o catálogo mude |
| `acquisition_source` | De qual canal veio (web, mobile) — útil para relatórios |
| `expires_at` | Para pedidos com pagamento a vencer (ex.: Pix com 48h de expiração) |
| `coupon_code` | Registro do cupom usado, mesmo que o cupom seja deletado depois |

---

#### OrderItem

Snapshot imutável de cada item do pedido. **Não referencia o catálogo para valores** —
tudo é copiado no momento da criação.

```python
class OrderItem(models.Model):
    ITEM_TYPE_CHOICES = [
        ("product",           _("Product")),
        ("subscription_plan", _("Subscription Plan")),
    ]

    order             = models.ForeignKey(Order, on_delete=models.CASCADE, related_name="items")
    product           = models.ForeignKey("catalog.Product", on_delete=models.PROTECT,
                                          related_name="order_items", null=True, blank=True)
    subscription_plan = models.ForeignKey("subscriptions.SubscriptionPlan", on_delete=models.PROTECT,
                                          related_name="order_items", null=True, blank=True)
    item_type          = models.CharField(max_length=20, choices=ITEM_TYPE_CHOICES, default="product")
    title              = models.CharField(max_length=255)       # copiado do catálogo
    description        = models.TextField(blank=True)
    quantity           = models.PositiveIntegerField(default=1)
    unit_price         = models.DecimalField(max_digits=10, decimal_places=2)
    commission_percent = models.DecimalField("platform commission (%)", max_digits=5, decimal_places=2, default=10)
    platform_amount    = models.DecimalField("platform revenue", max_digits=10, decimal_places=2, default=0)
    producer_amount    = models.DecimalField("producer payout", max_digits=10, decimal_places=2, default=0)
    metadata           = models.JSONField(default=dict, blank=True)

    @property
    def line_total(self) -> Decimal:
        return self.unit_price * self.quantity
```

**Sobre comissões:**

`commission_percent` é copiado do produto/plano no momento do pedido e nunca muda.
`platform_amount` e `producer_amount` são calculados e gravados atomicamente — nunca
recalcule depois.

FK para `product`/`subscription_plan` usa `on_delete=PROTECT` para impedir deleção
de itens do catálogo que têm histórico de venda.

---

### 2.2 `payments/models.py`

#### PaymentMethod

Configuração de métodos de pagamento aceitos. Gerenciada pelo admin, sem código hardcoded.

```python
class PaymentMethod(models.Model):
    CODE_CHOICES = [
        ("pix",           "Pix"),
        ("card",          _("Credit Card")),
        ("bank_transfer", _("Bank Transfer")),
    ]

    code                    = models.CharField(max_length=20, choices=CODE_CHOICES, unique=True)
    name                    = models.CharField(max_length=80)
    is_active               = models.BooleanField(default=False)
    sort_order              = models.PositiveSmallIntegerField(default=0)
    instructions            = models.TextField(blank=True)
    pix_key                 = models.CharField(max_length=120, blank=True)
    pix_beneficiary_name    = models.CharField(max_length=120, blank=True)
    pix_beneficiary_document = models.CharField(max_length=20, blank=True)

    class Meta:
        ordering = ("sort_order",)
```

**Por que admin-configurável:** evita redeploys para ativar/desativar métodos de pagamento.
O admin pode habilitar "Pix" na véspera de um lançamento sem deploys.

---

#### PaymentTransaction

Registro de cada tentativa de pagamento associada a um pedido.

```python
class PaymentTransaction(models.Model):
    STATUS_CHOICES = [
        ("pending",    _("Pending")),
        ("authorized", _("Authorized")),
        ("paid",       _("Paid")),
        ("refused",    _("Refused")),
        ("refunded",   _("Refunded")),
    ]

    order              = models.ForeignKey("commerce.Order", on_delete=models.CASCADE,
                                           related_name="transactions")
    provider           = models.CharField(max_length=40, default="manual")
    method             = models.CharField(max_length=20, blank=True)
    external_reference = models.CharField(max_length=120, blank=True)
    status             = models.CharField(max_length=20, choices=STATUS_CHOICES, default="pending")
    amount             = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    payload            = models.JSONField(default=dict, blank=True)
    instructions       = models.TextField(blank=True)
    confirmed_by       = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.SET_NULL,
                                           related_name="confirmed_transactions", null=True, blank=True)
    confirmed_at       = models.DateTimeField(null=True, blank=True)
    paid_at            = models.DateTimeField(null=True, blank=True)
    expires_at         = models.DateTimeField(null=True, blank=True)
    created_at         = models.DateTimeField(auto_now_add=True)
```

**Campos importantes:**

| Campo | Propósito |
|-------|-----------|
| `provider` | `"manual"` para confirmação humana; `"stripe"`, `"pagarme"`, etc. para gateways |
| `external_reference` | ID do pedido ou ID gerado pelo gateway — usado para correlação |
| `payload` | JSON livre: chave Pix, QR code, resposta do gateway, etc. |
| `instructions` | Texto copiado do `PaymentMethod` no momento do pedido |
| `confirmed_by` | Quem confirmou (staff user ou `None` se confirmado por webhook) |

---

#### WebhookEvent

Tabela de deduplicação para webhooks de gateways externos. Evita processar o mesmo evento
duas vezes em caso de retentativas do gateway.

```python
class WebhookEvent(models.Model):
    provider    = models.CharField(max_length=40)
    event_type  = models.CharField(max_length=120)
    external_id = models.CharField(max_length=120, unique=True)   # chave de idempotência
    processed   = models.BooleanField(default=False)
    payload     = models.JSONField(default=dict, blank=True)
    created_at  = models.DateTimeField(auto_now_add=True)
```

**Uso:**
```python
event, created = WebhookEvent.objects.get_or_create(
    external_id=webhook_id,
    defaults={"provider": "stripe", "event_type": event_type, "payload": data},
)
if not created:
    return  # já processado
# ... processa ...
event.processed = True
event.save()
```

---

## 3. Serviços

### 3.1 `commerce/services.py` — Criação do Pedido

#### `calculate_commission_amounts()`

```python
from decimal import Decimal, ROUND_HALF_UP

MONEY = Decimal("0.01")

def calculate_commission_amounts(
    line_total: Decimal, commission_percent: Decimal
) -> tuple[Decimal, Decimal]:
    platform_amount = (
        line_total * commission_percent / Decimal("100")
    ).quantize(MONEY, rounding=ROUND_HALF_UP)
    producer_amount = (line_total - platform_amount).quantize(MONEY, rounding=ROUND_HALF_UP)
    return platform_amount, producer_amount
```

**Regra:** arredonda com `ROUND_HALF_UP` e garante que `platform + producer == line_total`
subtraindo em vez de arredondar os dois lados separadamente.

---

#### `create_order_from_cart()`

Função principal do checkout. Deve ser executada dentro de uma transação de banco de dados.

```python
from django.db import transaction

@transaction.atomic
def create_order_from_cart(user, cart, payment_method_code: str, source: str = "web") -> Order:
    items = list(cart.items.select_related("product", "subscription_plan").all())
    if not items:
        raise ValueError("Cart is empty.")

    # 1. Calcular total e montar snapshot
    snapshot_items = []
    total = Decimal("0.00")
    for item in items:
        snapshot_items.append({
            "title":      str(item.display_title),
            "type":       str(item.display_type),
            "quantity":   item.quantity,
            "unit_price": str(item.unit_price),
            "line_total": str(item.line_total),
        })
        total += item.line_total

    # 2. Criar Order
    order = Order.objects.create(
        user=user,
        status="pending",
        total_amount=total,
        payment_method=payment_method_code,
        acquisition_source=source,
        checkout_snapshot={"items": snapshot_items, "total": str(total)},
    )

    # 3. Copiar itens do carrinho como OrderItems imutáveis
    for item in items:
        is_plan = bool(item.subscription_plan_id)
        obj = item.subscription_plan if is_plan else item.product
        commission_percent = obj.platform_commission_percent
        platform_amount, producer_amount = calculate_commission_amounts(
            item.line_total, commission_percent
        )
        OrderItem.objects.create(
            order=order,
            product=item.product if not is_plan else None,
            subscription_plan=item.subscription_plan if is_plan else None,
            item_type="subscription_plan" if is_plan else "product",
            title=item.display_title,
            quantity=item.quantity,
            unit_price=item.unit_price,
            commission_percent=commission_percent,
            platform_amount=platform_amount,
            producer_amount=producer_amount,
            metadata={
                "slug": obj.slug,
                "commission_percent": str(commission_percent),
                "platform_amount": str(platform_amount),
                "producer_amount": str(producer_amount),
            },
        )

    # 4. Criar PaymentTransaction
    _create_transaction(order, payment_method_code, total)

    # 5. Limpar carrinho
    cart.items.all().delete()

    # 6. Enviar e-mail de resumo do pedido
    _send_order_summary_email(order)

    return order
```

**Sequência obrigatória:** Order → OrderItems → PaymentTransaction → limpar carrinho →
e-mail. Se `@transaction.atomic` for usado, o e-mail deve ser disparado **fora** da
transação (ou usando `transaction.on_commit()`), pois falha de e-mail não deve reverter
o pedido.

---

#### `_create_transaction()`

Cria o registro de transação de pagamento. Para Pix, preenche a chave e o vencimento.

```python
def _create_transaction(order: Order, method_code: str, amount: Decimal):
    from apps.payments.models import PaymentMethod, PaymentTransaction

    method_cfg = PaymentMethod.objects.filter(code=method_code).first()
    expires_at = None
    instructions = method_cfg.instructions if method_cfg else ""
    payload = {}

    if method_code == "pix":
        expires_at = timezone.now() + timedelta(hours=48)
        if method_cfg:
            payload = {
                "pix_key":                   method_cfg.pix_key,
                "pix_beneficiary_name":      method_cfg.pix_beneficiary_name,
                "pix_beneficiary_document":  method_cfg.pix_beneficiary_document,
            }

    PaymentTransaction.objects.create(
        order=order,
        provider="manual",
        method=method_code,
        status="pending",
        amount=amount,
        instructions=instructions,
        expires_at=expires_at,
        payload=payload,
        external_reference=f"order-{order.pk}",
    )
```

---

### 3.2 `payments/services.py` — Confirmação de Pagamento

#### `confirm_payment()`

Confirma uma transação e concede acesso. **Idempotente:** se já `paid`, retorna
imediatamente sem efeitos colaterais.

```python
def confirm_payment(transaction, confirmed_by) -> None:
    if transaction.status == "paid":
        return

    now = timezone.now()
    transaction.status       = "paid"
    transaction.paid_at      = now
    transaction.confirmed_by = confirmed_by
    transaction.confirmed_at = now
    transaction.save(update_fields=["status", "paid_at", "confirmed_by", "confirmed_at"])

    order = transaction.order
    order.status  = "paid"
    order.paid_at = now
    order.save(update_fields=["status", "paid_at"])

    grant_order_access(order)
    _send_access_granted_email(order)
```

**Usar `update_fields`** em ambos os saves para evitar race conditions com outros campos
sendo gravados em paralelo.

---

#### `grant_order_access()`

Itera os OrderItems do pedido e chama o handler adequado para cada tipo.

```python
BILLING_DAYS = {
    "monthly":    31,
    "semiannual": 183,
    "yearly":     365,
}

def grant_order_access(order) -> None:
    now = timezone.now()
    for item in order.items.select_related("product", "subscription_plan").all():
        if item.subscription_plan_id:
            _grant_subscription(order, item.subscription_plan, now)
        elif item.product_id:
            _grant_product(order, item.product)
```

#### `_grant_subscription()`

```python
def _grant_subscription(order, plan, now):
    from apps.subscriptions.models import Subscription
    from apps.library.models import Entitlement

    days    = BILLING_DAYS.get(plan.billing_interval, 31)
    ends_at = now + timedelta(days=days)
    idempotency_ref = f"order-{order.pk}-plan-{plan.pk}"

    sub, created = Subscription.objects.get_or_create(
        user=order.user,
        plan=plan,
        external_reference=idempotency_ref,   # garante idempotência
        defaults={"status": "active", "starts_at": now, "ends_at": ends_at},
    )

    if created:
        Entitlement.objects.create(
            user=order.user,
            subscription=sub,
            source="subscription",
            active=True,
            expires_at=ends_at,
        )
```

#### `_grant_product()`

```python
def _grant_product(order, product):
    from apps.library.models import Entitlement, LibraryEntry

    ent, created = Entitlement.objects.get_or_create(
        user=order.user,
        product=product,
        source="purchase",
        defaults={"active": True},
    )

    if created:
        LibraryEntry.objects.get_or_create(
            user=order.user,
            product=product,
            defaults={"entitlement": ent},
        )
```

**Idempotência em ambos os casos:** `get_or_create` com campos que identificam unicamente
o acesso. Reprocessar o mesmo pedido não duplica entitlements.

---

## 4. Views

### 4.1 `commerce/views.py`

#### `_get_or_create_cart(request)` — helper interno

Centraliza a lógica de obtenção do carrinho. Trata usuários autenticados, sessões
anônimas e a mesclagem ao fazer login.

```python
def _get_or_create_cart(request):
    if request.user.is_authenticated:
        cart, _ = Cart.objects.get_or_create(user=request.user, defaults={"session_key": ""})
        # Mesclar carrinho de sessão anônima se existir
        session_key = request.session.session_key
        if session_key:
            session_cart = Cart.objects.filter(session_key=session_key, user=None).first()
            if session_cart:
                for item in session_cart.items.select_related("product", "subscription_plan").all():
                    if item.subscription_plan_id:
                        # Plano: substitui (apenas 1 por vez)
                        cart.items.filter(subscription_plan__isnull=False).delete()
                        item.cart = cart
                        item.save()
                    else:
                        existing = cart.items.filter(product=item.product).first()
                        if existing:
                            existing.quantity += item.quantity
                            existing.save()
                        else:
                            item.cart = cart
                            item.save()
                session_cart.delete()
        return cart
    else:
        if not request.session.session_key:
            request.session.create()
        return Cart.objects.get_or_create(
            session_key=request.session.session_key,
            user=None,
            defaults={},
        )[0]
```

---

#### `cart_detail(request)` — GET `/cart/`

Exibe o carrinho. Também aceita `?add=<product_id>` para adicionar produto via GET (link
direto de "Adicionar ao carrinho").

```python
def cart_detail(request):
    add_id = request.GET.get("add")
    if add_id:
        try:
            product = Product.objects.get(pk=add_id, is_published=True)
            cart = _get_or_create_cart(request)
            item, created = CartItem.objects.get_or_create(
                cart=cart, product=product,
                defaults={"unit_price": product.primary_price, "quantity": 1},
            )
            if not created:
                item.quantity += 1
                item.save()
            messages.success(request, f'"{product.title}" added to cart.')
        except Product.DoesNotExist:
            messages.error(request, "Product not found.")
        return redirect("commerce:cart_detail")

    # Buscar carrinho para exibição
    cart = Cart.objects.filter(user=request.user) \
               .prefetch_related("items__product", "items__subscription_plan") \
               .order_by("-updated_at").first() \
        if request.user.is_authenticated else ...

    return render(request, "commerce/cart.html", {"cart": cart})
```

---

#### `checkout(request)` — POST `/checkout/`

Requer login. Valida o método de pagamento, chama `create_order_from_cart()` e redireciona.

```python
@login_required_redirect  # ou: redirecionar manualmente com ?next=
def checkout(request):
    cart = Cart.objects.filter(user=request.user) \
               .prefetch_related("items__product", "items__subscription_plan") \
               .order_by("-updated_at").first()

    if not cart or not cart.items.exists():
        messages.warning(request, "Your cart is empty.")
        return redirect("commerce:cart_detail")

    payment_methods = list(PaymentMethod.objects.filter(is_active=True).order_by("sort_order"))

    if request.method == "POST":
        method_code  = request.POST.get("payment_method", "").strip()
        active_codes = {m.code for m in payment_methods}

        if not method_code or method_code not in active_codes:
            messages.error(request, "Select a valid payment method.")
            return render(request, "commerce/checkout.html", {
                "cart": cart, "payment_methods": payment_methods
            })

        try:
            order = create_order_from_cart(request.user, cart, method_code)
        except ValueError as exc:
            messages.error(request, str(exc))
            return redirect("commerce:cart_detail")

        if method_code in {"pix", "bank_transfer"}:
            return redirect("commerce:payment_instructions", order_id=order.pk)
        return redirect("commerce:order_detail", order_id=order.pk)

    return render(request, "commerce/checkout.html", {
        "cart": cart, "payment_methods": payment_methods
    })
```

---

#### `payment_instructions(request, order_id)` — GET `/order/<id>/payment/`

Exibe as instruções de pagamento (chave Pix, dados bancários, etc.) para pagamentos
manuais. Seleciona o template com base no método.

```python
def payment_instructions(request, order_id: int):
    order       = get_object_or_404(Order, pk=order_id, user=request.user)
    transaction = order.transactions.filter(method=order.payment_method).order_by("-created_at").first()
    payment_cfg = PaymentMethod.objects.filter(code=order.payment_method).first()

    template = "commerce/payment_pix.html" if order.payment_method == "pix" \
               else "commerce/payment_manual.html"

    return render(request, template, {
        "order":       order,
        "transaction": transaction,
        "payment_cfg": payment_cfg,
        "is_pix":      order.payment_method == "pix",
    })
```

---

### 4.2 `commerce/urls.py`

```python
from django.urls import path
from . import views

app_name = "commerce"

urlpatterns = [
    path("",                            views.cart_detail,          name="cart_detail"),
    path("remove/<int:item_id>/",       views.remove_from_cart,     name="remove_from_cart"),
    path("checkout/",                   views.checkout,             name="checkout"),
    path("order/<int:order_id>/payment/", views.payment_instructions, name="payment_instructions"),
    path("orders/",                     views.order_list,           name="order_list"),
    path("orders/<int:order_id>/",      views.order_detail,         name="order_detail"),
]
```

---

## 5. Sistema de E-mail Transacional

O sistema separa **configuração SMTP** (admin-configurável), **templates de e-mail**
(editáveis via admin, com suporte a HTML e texto), e **log de entregas** (com idempotência).

### 5.1 `notifications/models.py`

#### EmailConfiguration

SMTP configurável pelo admin, sem variáveis de ambiente expostas.

```python
class EmailConfiguration(models.Model):
    name              = models.CharField(max_length=100)
    is_active         = models.BooleanField(default=True)
    backend           = models.CharField(max_length=255,
                            default="django.core.mail.backends.smtp.EmailBackend")
    host              = models.CharField(max_length=255, blank=True)
    port              = models.PositiveIntegerField(default=587)
    username          = models.CharField(max_length=255, blank=True)
    password          = models.CharField(max_length=255, blank=True)
    use_tls           = models.BooleanField(default=True)
    use_ssl           = models.BooleanField(default=False)
    timeout           = models.PositiveIntegerField(default=15)
    default_from_email = models.CharField(max_length=254, blank=True)
    default_reply_to  = models.EmailField(blank=True)
    updated_at        = models.DateTimeField(auto_now=True)

    def clean(self):
        if self.use_tls and self.use_ssl:
            raise ValidationError("Choose TLS or SSL, not both.")

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        if self.is_active:
            # Garante apenas uma configuração ativa por vez
            EmailConfiguration.objects.exclude(pk=self.pk).update(is_active=False)
```

#### EmailTemplate

Templates com suporte a Django template language (variáveis, condicionais, filtros).

```python
class EmailTemplate(models.Model):
    key       = models.SlugField(max_length=50, unique=True)  # ex: "order_summary"
    name      = models.CharField(max_length=100)
    subject   = models.CharField(max_length=255)              # suporta {{ variáveis }}
    preheader = models.CharField(max_length=255, blank=True)
    body_text = models.TextField()                            # versão plain-text
    body_html = models.TextField(blank=True)                  # versão HTML
    from_email = models.CharField(max_length=254, blank=True)
    reply_to  = models.EmailField(blank=True)
    is_active = models.BooleanField(default=True)
```

#### EmailDeliveryLog

Registro auditável de todos os e-mails enviados ou falhos.

```python
class EmailDeliveryLog(models.Model):
    STATUS_CHOICES = [
        ("pending", "Pending"), ("sent", "Sent"),
        ("failed",  "Failed"),  ("skipped", "Skipped"),
    ]

    template          = models.ForeignKey(EmailTemplate, on_delete=models.SET_NULL,
                                          null=True, blank=True, related_name="logs")
    recipient_email   = models.EmailField()
    user              = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.SET_NULL,
                                          null=True, blank=True, related_name="email_logs")
    subject_rendered  = models.CharField(max_length=255)
    status            = models.CharField(max_length=20, choices=STATUS_CHOICES, default="pending")
    idempotency_key   = models.CharField(max_length=255, blank=True, db_index=True)
    context_snapshot  = models.JSONField(default=dict, blank=True)  # debug
    error_message     = models.TextField(blank=True)
    sent_at           = models.DateTimeField(null=True, blank=True)
    created_at        = models.DateTimeField(auto_now_add=True)
```

---

### 5.2 `notifications/services.py`

#### `send_transactional_email()`

Função central de envio. Checa idempotência, renderiza template, envia e registra log.

```python
def send_transactional_email(
    key: str,
    recipient,           # User instance ou string de e-mail
    context: dict,
    idempotency_key: str = "",
) -> bool:
    from .models import EmailDeliveryLog

    # Resolve destinatário
    User = get_user_model()
    if isinstance(recipient, User):
        to_email = recipient.email
        user     = recipient
    else:
        to_email = str(recipient)
        user     = None

    # Idempotência: não reenvia se já enviado com sucesso
    if idempotency_key and EmailDeliveryLog.objects.filter(
        idempotency_key=idempotency_key, status="sent"
    ).exists():
        return False

    # Renderiza template
    rendered = render_email_template(key, context)
    if rendered is None:
        return False  # template não existe ou inativo — falha silenciosa

    # Cria log antes de enviar (status=pending)
    log = EmailDeliveryLog.objects.create(
        template=rendered["template"],
        recipient_email=to_email,
        user=user,
        subject_rendered=rendered["subject"],
        idempotency_key=idempotency_key,
        context_snapshot=_safe_snapshot(context),
    )

    try:
        msg = EmailMultiAlternatives(
            subject=rendered["subject"],
            body=rendered["body_text"],
            from_email=rendered["from_email"],
            to=[to_email],
            reply_to=rendered["reply_to"] or None,
            connection=get_email_connection(rendered["email_config"]),
        )
        if rendered["body_html"]:
            msg.attach_alternative(rendered["body_html"], "text/html")
        msg.send()
        log.status  = "sent"
        log.sent_at = timezone.now()
        log.save(update_fields=["status", "sent_at"])
        return True

    except Exception:
        logger.exception("Failed to send email key=%s to=%s", key, to_email)
        log.status        = "failed"
        log.error_message = str(exc)
        log.save(update_fields=["status", "error_message"])
        return False
```

#### `render_email_template()`

```python
def render_email_template(key: str, context: dict) -> dict | None:
    try:
        tmpl = EmailTemplate.objects.get(key=key, is_active=True)
    except EmailTemplate.DoesNotExist:
        logger.warning("EmailTemplate key=%s not found or inactive.", key)
        return None

    text_ctx = Context(context, autoescape=False)
    html_ctx = Context(context)

    email_config = get_active_email_configuration()
    return {
        "subject":    Template(tmpl.subject).render(text_ctx),
        "body_text":  Template(tmpl.body_text).render(text_ctx),
        "body_html":  Template(tmpl.body_html).render(html_ctx) if tmpl.body_html else "",
        "from_email": tmpl.from_email or get_default_from_email(email_config),
        "reply_to":   [tmpl.reply_to] if tmpl.reply_to else get_default_reply_to(email_config),
        "template":   tmpl,
        "email_config": email_config,
    }
```

---

### 5.3 Templates de E-mail necessários

Criar via Django admin (`/admin/notifications/emailtemplate/`) com as seguintes chaves:

| Key | Quando é enviado | Contexto disponível |
|-----|-----------------|---------------------|
| `order_summary` | No checkout, após criação do pedido | `user`, `order`, `items`, `site`, `support_email`, `library_url` |
| `access_granted` | Após confirmação do pagamento | `user`, `order`, `items`, `site`, `support_email`, `library_url` |
| `welcome` | Após registro do usuário | `user`, `site` |
| `email_confirmation` | Verificação de e-mail (allauth) | padrão do allauth |

---

## 6. Admin

### 6.1 `commerce/admin.py` — pontos-chave

```python
@admin.register(Order)
class OrderAdmin(ModelAdmin):
    list_display   = ("id", "user_email", "status_badge", "payment_method",
                      "total_amount", "platform_total", "producer_total", "paid_at", "created_at")
    list_filter    = ("status", "payment_method", "acquisition_source",
                      ("created_at", admin.DateFieldListFilter))
    date_hierarchy = "created_at"
    search_fields  = ("user__email", "coupon_code")
    readonly_fields = ("checkout_snapshot", "paid_at", "created_at", "total_amount")
    actions        = [action_confirm_payment]
    inlines        = [OrderItemInline, PaymentTransactionInline]
```

**Ação de confirmação via admin:**

```python
def action_confirm_payment(modeladmin, request, queryset):
    from apps.payments.services import confirm_payment
    confirmed = 0
    for order in queryset.filter(status="pending"):
        transaction = order.transactions.filter(status="pending").order_by("-created_at").first()
        if transaction:
            confirm_payment(transaction, confirmed_by=request.user)
            confirmed += 1
    messages.success(request, f"{confirmed} order(s) confirmed and access granted.")
action_confirm_payment.short_description = "Confirm payment and grant access"
```

**Impedir deleção de pedidos pagos:**

```python
def has_delete_permission(self, request, obj=None):
    if obj and obj.status == "paid":
        return False
    return super().has_delete_permission(request, obj)
```

### 6.2 `payments/admin.py` — ação de confirmação direta na transação

Mesma lógica, mas operando diretamente sobre `PaymentTransaction`:

```python
def action_confirm_payment(modeladmin, request, queryset):
    from apps.payments.services import confirm_payment
    for transaction in queryset.filter(status="pending"):
        confirm_payment(transaction, confirmed_by=request.user)
```

---

## 7. Métricas (`commerce/metrics.py`)

Funções puras de consulta — sem efeitos colaterais.

```python
def revenue_summary(start=None, end=None) -> dict:
    """Resumo financeiro de um período. Padrão: mês corrente."""
    if start is None or end is None:
        start, end = current_month_period()

    paid_orders = Order.objects.filter(status="paid", paid_at__gte=start, paid_at__lte=end)
    paid_items  = OrderItem.objects.filter(order__status="paid",
                                           order__paid_at__gte=start, order__paid_at__lte=end)

    order_agg = paid_orders.aggregate(
        gross=Coalesce(Sum("total_amount"), ZERO, output_field=DecimalField()),
        count=Count("id"),
    )
    item_agg = paid_items.aggregate(
        platform=Coalesce(Sum("platform_amount"), ZERO, output_field=DecimalField()),
        producer=Coalesce(Sum("producer_amount"), ZERO, output_field=DecimalField()),
    )

    gross = order_agg["gross"]
    count = order_agg["count"]
    return {
        "gross_revenue":       gross,
        "platform_commission": item_agg["platform"],
        "producer_payout":     item_agg["producer"],
        "paid_orders_count":   count,
        "average_ticket":      gross / count if count else ZERO,
    }


def product_ranking(start=None, end=None, limit=5):
    """Produtos mais vendidos no período, com receita e comissão."""
    line_total = ExpressionWrapper(F("unit_price") * F("quantity"),
                                   output_field=DecimalField())
    return (
        paid_items_for_period(start, end)
        .filter(product__isnull=False)
        .values("product_id", "title")
        .annotate(
            sold_quantity=Sum("quantity"),
            gross_revenue=Sum(line_total),
            platform_commission=Sum("platform_amount"),
        )
        .order_by("-sold_quantity", "-gross_revenue")[:limit]
    )


def pending_orders_alerts(now=None) -> dict:
    """Contagem de pedidos pendentes há mais de 24h e 48h."""
    now = now or timezone.now()
    return {
        "pending_24h_count": Order.objects.filter(
            status="pending", created_at__lt=now - timedelta(hours=24)
        ).count(),
        "pending_48h_count": Order.objects.filter(
            status="pending", created_at__lt=now - timedelta(hours=48)
        ).count(),
    }
```

---

## 8. Configuração do Projeto

### 8.1 `settings.py`

```python
INSTALLED_APPS = [
    # ...
    "django.contrib.sites",        # necessário para Site.objects.get_current()
    "apps.commerce",
    "apps.payments",
    "apps.notifications",
    # ...
]

SITE_ID = 1
```

### 8.2 `config/urls.py`

```python
from django.urls import include, path

urlpatterns = [
    path("cart/",    include("apps.commerce.urls")),
    path("",         include("apps.cms.urls")),        # homepage e páginas estáticas
    path("admin/",   admin.site.urls),
]
```

### 8.3 Migrations necessárias

```bash
python manage.py makemigrations commerce payments notifications
python manage.py migrate
python manage.py createcachetable   # se usar cache-based sessions
```

---

## 9. Diagrama ER Simplificado

```
Cart ──< CartItem >── Product
                 >── SubscriptionPlan

Order ──< OrderItem >── Product
     │              >── SubscriptionPlan
     └──< PaymentTransaction

EmailTemplate ──< EmailDeliveryLog >── User

Subscription >── SubscriptionPlan
Entitlement  >── Product
             >── Subscription
LibraryEntry >── Product
             >── Entitlement
```

---

## 10. Fluxo Completo Passo a Passo

### 10.1 Adicionar ao carrinho

1. `GET /cart/?add=<product_id>` chama `_get_or_create_cart(request)`.
2. `CartItem.objects.get_or_create()` — incrementa quantidade se já existe.
3. Redireciona para `GET /cart/` (exibe o carrinho).

### 10.2 Checkout

1. `GET /checkout/` — exibe lista de métodos ativos + resumo do carrinho.
2. `POST /checkout/` com `payment_method=pix`.
3. `create_order_from_cart()` cria `Order (pending)`, `OrderItem`s e `PaymentTransaction`.
4. E-mail `order_summary` enviado.
5. Redireciona para `/order/<id>/payment/` (instruções Pix).

### 10.3 Confirmação manual (admin)

1. Admin acessa `/admin/payments/paymenttransaction/`.
2. Seleciona transação → ação "Confirm payment and grant access".
3. `confirm_payment(transaction, confirmed_by=request.user)`:
   - `transaction.status = "paid"`.
   - `order.status = "paid"`.
   - `grant_order_access(order)` cria `Subscription` ou `Entitlement`.
   - E-mail `access_granted` enviado.

### 10.4 Webhook de gateway externo

```python
# apps/payments/views.py
def stripe_webhook(request):
    payload   = request.body
    sig_header = request.META.get("HTTP_STRIPE_SIGNATURE")
    try:
        event = stripe.Webhook.construct_event(payload, sig_header, settings.STRIPE_WEBHOOK_SECRET)
    except (ValueError, stripe.error.SignatureVerificationError):
        return HttpResponse(status=400)

    # Idempotência via WebhookEvent
    webhook_event, created = WebhookEvent.objects.get_or_create(
        external_id=event["id"],
        defaults={"provider": "stripe", "event_type": event["type"], "payload": event},
    )
    if not created:
        return HttpResponse(status=200)  # já processado

    if event["type"] == "payment_intent.succeeded":
        payment_intent_id = event["data"]["object"]["id"]
        transaction = PaymentTransaction.objects.filter(
            external_reference=payment_intent_id
        ).first()
        if transaction:
            confirm_payment(transaction, confirmed_by=None)

    webhook_event.processed = True
    webhook_event.save()
    return HttpResponse(status=200)
```

---

## 11. Decisões de Design e Tradeoffs

| Decisão | Motivação | Tradeoff |
|---------|-----------|----------|
| Snapshot JSON no pedido | Preserva o estado no momento da compra | Redundância com o catálogo |
| `unit_price` no CartItem | Garante que mudanças de preço não afetam carrinhos ativos | Preço pode ficar desatualizado no carrinho |
| `commission_percent` gravado no OrderItem | Comissão histórica é imutável | Não reflete mudanças futuras de comissão |
| `get_or_create` com `external_reference` | Idempotência nativa na concessão de acesso | Precisa de campo adicional na FK |
| `on_delete=PROTECT` em OrderItem → Product | Impede deleção acidental de catálogo com histórico | Precisa marcar como inativo em vez de deletar |
| `PaymentMethod` configurável via admin | Zero deploys para mudar métodos aceitos | Lógica de negócio parcialmente no banco |
| E-mail enviado fora de `@transaction.atomic` | Falha de e-mail não reverte o pedido | Pedido pode existir sem e-mail enviado |
| `EmailDeliveryLog` + `idempotency_key` | Evita reenvio em caso de retry | Custo de query a cada envio |

---

## 12. Checklist de Implementação

### Modelos
- [ ] `Cart` com suporte a usuário autenticado e sessão anônima
- [ ] `CartItem` com `UniqueConstraint` condicional por tipo de item
- [ ] `Order` com `checkout_snapshot` (JSONField), `acquisition_source`, `expires_at`
- [ ] `OrderItem` com `commission_percent`, `platform_amount`, `producer_amount` (imutáveis)
- [ ] `PaymentMethod` admin-configurável com campos Pix
- [ ] `PaymentTransaction` com `external_reference`, `payload`, `confirmed_by`
- [ ] `WebhookEvent` com `external_id` único (idempotência)
- [ ] `EmailTemplate`, `EmailConfiguration`, `EmailDeliveryLog`

### Serviços
- [ ] `calculate_commission_amounts()` com `ROUND_HALF_UP`
- [ ] `create_order_from_cart()` — atômico, com snapshot e limpeza do carrinho
- [ ] `_create_transaction()` separado por método de pagamento
- [ ] `confirm_payment()` — idempotente via guard `if status == "paid": return`
- [ ] `grant_order_access()` — usando `get_or_create` com campo de idempotência
- [ ] `send_transactional_email()` — com deduplicação por `idempotency_key`

### Views
- [ ] `_get_or_create_cart()` — mesclagem de carrinho anônimo ao logar
- [ ] `cart_detail` — suporte a `?add=<id>` para adicionar via GET
- [ ] `checkout` — valida método ativo antes de criar pedido
- [ ] `payment_instructions` — template condicional por método
- [ ] `order_list` e `order_detail` — apenas para o próprio usuário

### Admin
- [ ] Ação "Confirmar pagamento" em `Order` e em `PaymentTransaction`
- [ ] `has_delete_permission` bloqueia deleção de pedidos `paid`
- [ ] `OrderItemInline` e `PaymentTransactionInline` em `OrderAdmin`
- [ ] Status colorido (`format_html`) em listagens

### URLs
- [ ] `cart/` → `commerce.urls`
- [ ] Namespace `app_name = "commerce"`

### E-mail
- [ ] Template `order_summary` cadastrado no admin
- [ ] Template `access_granted` cadastrado no admin
- [ ] `EmailConfiguration` ativa com SMTP correto

### Testes a validar manualmente
- [ ] Adicionar produto ao carrinho sem login → fazer login → carrinho mesclado
- [ ] Checkout com carrinho vazio → redireciona com aviso
- [ ] Checkout com método inativo → erro correto
- [ ] Confirmar pagamento duas vezes → segundo `confirm_payment()` é no-op
- [ ] Webhook duplicado → `WebhookEvent` existe, retorna 200 sem reprocessar
- [ ] E-mail com mesma `idempotency_key` → segundo envio ignorado
