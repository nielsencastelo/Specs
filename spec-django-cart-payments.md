# Spec: Shopping Cart and Payment System in Django

> Generic reference document. Describes how to implement a complete system for
> cart, checkout, and payments in any Django project. It is not business-specific.
> An AI can read this document and implement the system from scratch.

---

## 1. Architecture Overview

The system is divided into two Django apps:

| App | Responsibility |
|-----|-----------------|
| `commerce` | Cart, orders, order items, commissions, metrics |
| `payments` | Payment methods, transactions, webhooks, confirmation |

A third app (`notifications`) handles transactional emails with support for
editable templates via admin and idempotency using unique keys.

### High-level Flow

```
[user] → adds item to cart
       → cart_detail (GET /cart/)
       → checkout POST → create_order_from_cart()
             ↓
       Order (pending) + OrderItems + PaymentTransaction (pending)
             ↓
       [user pays manually: Pix / transfer]
             ↓
       [admin confirms] → confirm_payment()
             ↓
       Order (paid) + grant_order_access() + access granted email
```

---

## 2. Models

### 2.1 `commerce/models.py`

#### Cart

Represents the active cart of a user or an anonymous session.

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

**Rules:**
- `user` and `session_key` are practically mutually exclusive.
- A user can have only one active cart. Anonymous carts are identified by `session_key`.
- When an anonymous user logs in, the session cart is merged into the user's cart.

---

#### CartItem

Each line in the cart. Supports two types of items: single product **or** subscription plan.

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

    @property
    def line_total(self) -> Decimal:
        return self.unit_price * self.quantity
```

---

#### Order

Immutable order created from the cart at checkout.

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

    user               = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                                           related_name="orders")
    status             = models.CharField(max_length=20, choices=STATUS_CHOICES, default="pending")
    total_amount       = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    payment_method     = models.CharField(max_length=20, choices=PAYMENT_METHOD_CHOICES, blank=True)
    checkout_snapshot  = models.JSONField(default=dict, blank=True)
    created_at         = models.DateTimeField(auto_now_add=True)
```

---

#### OrderItem

Immutable snapshot of each order item. **Does not reference the catalog for values** —
everything is copied at creation.

```python
class OrderItem(models.Model):
    order             = models.ForeignKey(Order, on_delete=models.CASCADE, related_name="items")
    product           = models.ForeignKey("catalog.Product", on_delete=models.PROTECT, null=True, blank=True)
    title              = models.CharField(max_length=255)       # copied from catalog
    quantity           = models.PositiveIntegerField(default=1)
    unit_price         = models.DecimalField(max_digits=10, decimal_places=2)
    commission_percent = models.DecimalField(max_digits=5, decimal_places=2, default=10)
    platform_amount    = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    producer_amount    = models.DecimalField(max_digits=10, decimal_places=2, default=0)
```

---

## 3. Services

### 3.1 `commerce/services.py` — Order Creation

#### `create_order_from_cart()`

Main checkout function. Must be executed within a database transaction.

```python
@transaction.atomic
def create_order_from_cart(user, cart, payment_method_code: str) -> Order:
    items = list(cart.items.all())
    # 1. Create Order
    order = Order.objects.create(
        user=user,
        status="pending",
        total_amount=cart.total,
        payment_method=payment_method_code,
    )
    # 2. Copy items to OrderItems
    # 3. Clear cart
    cart.items.all().delete()
    return order
```

---

### 3.2 `payments/services.py` — Payment Confirmation

#### `confirm_payment()`

Confirms a transaction and grants access. **Idempotent.**

```python
def confirm_payment(transaction, confirmed_by) -> None:
    if transaction.status == "paid":
        return
    transaction.status = "paid"
    transaction.save()
    order = transaction.order
    order.status = "paid"
    order.save()
    grant_order_access(order)
```

---

## 5. Transactional Email System

### 5.1 `notifications/models.py`

#### EmailTemplate

Templates with Django template language support.

```python
class EmailTemplate(models.Model):
    key       = models.SlugField(max_length=50, unique=True)  # e.g., "order_summary"
    subject   = models.CharField(max_length=255)
    body_html = models.TextField(blank=True)
    is_active = models.BooleanField(default=True)
```

---

## 11. Design Decisions and Tradeoffs

| Decision | Motivation | Tradeoff |
|---------|-----------|----------|
| JSON Snapshot in Order | Preserves state at the moment of purchase | Redundancy with catalog |
| `unit_price` in CartItem | Ensures price changes don't affect active carts | Price may become outdated in cart |
| `on_delete=PROTECT` | Prevents accidental deletion of catalog items with history | Must mark as inactive instead of deleting |

---

## 12. Implementation Checklist

- [ ] `Cart` supports authenticated users and anonymous sessions.
- [ ] `Order` with `checkout_snapshot` (JSONField).
- [ ] `OrderItem` with immutable commission and payouts.
- [ ] `PaymentTransaction` with idempotency logic.
- [ ] `confirm_payment()` is idempotent.
- [ ] `grant_order_access()` handles products and subscriptions.
- [ ] Email templates for `order_summary` and `access_granted`.
