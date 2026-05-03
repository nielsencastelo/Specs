# Internacionalização Django — Spec de Implementação

Guia técnico completo e genérico para implementar suporte a múltiplos idiomas em projetos Django. Cobre configuração, templates, modelos, views, troca de idioma por interface, e fluxo de tradução. Projetado para ser executado por uma IA como especificação de implementação.

---

## Índice

1. [Decisões Arquiteturais](#1-decisões-arquiteturais)
2. [Settings](#2-settings)
3. [Middleware](#3-middleware)
4. [URLs](#4-urls)
5. [Estrutura de Arquivos de Tradução](#5-estrutura-de-arquivos-de-tradução)
6. [Templates](#6-templates)
7. [Seletor de Idioma no Navbar](#7-seletor-de-idioma-no-navbar)
8. [Models](#8-models)
9. [Views](#9-views)
10. [Formulários](#10-formulários)
11. [Fluxo de Tradução](#11-fluxo-de-tradução)
12. [Checklist de Implementação](#12-checklist-de-implementação)

---

## 1. Decisões Arquiteturais

Antes de implementar, defina estas três escolhas. Elas afetam todas as demais etapas.

### 1.1 Estratégia de detecção de idioma

**Opção A — Sessão/Cookie (sem prefixo na URL)**
O idioma é salvo na sessão do usuário. As URLs permanecem iguais para todos os idiomas (`/catalogo/` funciona em qualquer idioma).

- Vantagens: URLs limpas, sem redirecionamentos, implementação simples.
- Desvantagens: Links não são compartilháveis com idioma fixo; bots de busca indexam apenas no idioma padrão.
- Quando usar: aplicações SaaS, plataformas com login obrigatório, quando SEO multilíngue não é crítico.

**Opção B — Prefixo na URL (`i18n_patterns`)**
Cada idioma tem seu próprio prefixo de URL (`/en/catalog/`, `/es/catalogo/`).

- Vantagens: URLs compartilháveis com idioma; SEO por idioma; hreflang funciona corretamente.
- Desvantagens: requer redirects ao trocar idioma; links internos precisam usar `{% url %}` com `LocalePrefixPattern`.
- Quando usar: sites públicos, e-commerce com SEO multilíngue, conteúdo estático.

**Este spec descreve a Opção A (sessão/cookie)**, que é a mais direta de implementar. As diferenças para a Opção B estão indicadas em notas ao longo do documento.

### 1.2 Idioma padrão (língua-fonte)

Defina qual idioma será usado para escrever o código-fonte (as strings nos arquivos `.py` e templates). As strings ficam como estão para esse idioma; arquivos `.po` são criados apenas para os demais.

Exemplo: se o projeto é em português (`pt-br`), as strings no código estão em português e os arquivos de tradução são criados para `en` e `es`.

### 1.3 Idiomas suportados

Liste os idiomas antes de começar. Cada idioma precisa de:
- Um diretório `locale/<código>/LC_MESSAGES/`
- Um arquivo `django.po` traduzido
- O arquivo compilado `django.mo`

---

## 2. Settings

Edite o arquivo de settings principal do projeto (`config/settings/base.py` ou equivalente).

### 2.1 Importação necessária

```python
from django.utils.translation import gettext_lazy as _
```

Essa importação deve estar no topo do arquivo de settings para que os nomes dos idiomas em `LANGUAGES` sejam traduzíveis.

### 2.2 Configurações obrigatórias

```python
# Idioma padrão da aplicação (língua em que o código está escrito)
LANGUAGE_CODE = "pt-br"   # substitua conforme seu projeto: "en", "es", etc.

# Lista de idiomas suportados
# O primeiro elemento de cada tupla é o código do idioma (usado em URLs e sessão)
# O segundo é o nome exibido ao usuário (envolto em _() para ser traduzível)
LANGUAGES = [
    ("pt-br", _("Português (Brasil)")),
    ("en",    _("English")),
    ("es",    _("Español")),
]

# Diretório raiz onde ficam as pastas de tradução
# Deve ser um único diretório na raiz do projeto
LOCALE_PATHS = [BASE_DIR / "locale"]

# Ativa o sistema de tradução do Django
USE_I18N = True

# Ativa localização de datas, números e moedas
USE_L10N = True

# Ativa suporte a timezone
USE_TZ = True

# Timezone padrão do servidor
TIME_ZONE = "America/Sao_Paulo"   # ajuste para seu fuso horário
```

### 2.3 Notas sobre os códigos de idioma

Django aceita dois formatos de código de idioma:
- Com hífen: `"pt-br"`, `"zh-hans"` — usado em `LANGUAGES` e na sessão
- Com underscore: `pt_BR`, `zh_Hans` — usado nos nomes de diretórios dentro de `locale/`

Django converte automaticamente entre os dois formatos. Porém, para evitar confusão, adote uma convenção consistente nos nomes de diretórios. Recomendação:

| Código no `LANGUAGES` | Diretório `locale/` |
|---|---|
| `"pt-br"` | `locale/pt_BR/` |
| `"en"` | `locale/en/` |
| `"es"` | `locale/es/` |
| `"zh-hans"` | `locale/zh_Hans/` |

---

## 3. Middleware

### 3.1 Posição obrigatória

`LocaleMiddleware` deve ficar **depois** de `SessionMiddleware` e **antes** de `CommonMiddleware`. Essa ordem é crítica: o middleware de locale precisa da sessão para ler o idioma salvo, e o `CommonMiddleware` precisa que o idioma já esteja ativo para redirecionamentos corretos.

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",        # se usar WhiteNoise
    "django.contrib.sessions.middleware.SessionMiddleware",   # ← sessão primeiro
    "corsheaders.middleware.CorsMiddleware",             # se usar CORS
    "django.middleware.locale.LocaleMiddleware",         # ← locale AQUI
    "django.middleware.common.CommonMiddleware",         # ← common depois
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    # outros middlewares de terceiros...
]
```

### 3.2 Como o LocaleMiddleware detecta o idioma

O middleware tenta detectar o idioma do usuário nesta ordem de prioridade:

1. `request.session[LANGUAGE_SESSION_KEY]` — idioma salvo na sessão pelo `set_language`
2. Cookie `django_language` — idioma salvo em cookie
3. Header HTTP `Accept-Language` do navegador
4. `LANGUAGE_CODE` do settings — idioma padrão (fallback final)

---

## 4. URLs

### 4.1 Registrar as URLs de i18n (Opção A — sem prefixo)

Adicione uma linha no arquivo principal de URLs (`config/urls.py`):

```python
from django.urls import include, path

urlpatterns = [
    path("i18n/", include("django.conf.urls.i18n")),  # ← obrigatório
    # ... demais URLs do projeto
]
```

Isso expõe a view `set_language` no endpoint `POST /i18n/set_language/`, que é chamada pelo formulário de troca de idioma.

### 4.2 Opção B — Com prefixo na URL

Se optar por URLs prefixadas, substitua `urlpatterns` por:

```python
from django.conf.urls.i18n import i18n_patterns
from django.urls import include, path

urlpatterns = [
    path("i18n/", include("django.conf.urls.i18n")),
]

urlpatterns += i18n_patterns(
    path("catalogo/", include("apps.catalog.urls")),
    path("biblioteca/", include("apps.library.urls")),
    # ... demais URLs que devem ter prefixo de idioma
    prefix_default_language=False,  # /catalogo/ em vez de /pt-br/catalogo/
)
```

Com `prefix_default_language=False`, o idioma padrão não tem prefixo na URL, e os demais têm (`/en/catalog/`, `/es/catalogo/`).

---

## 5. Estrutura de Arquivos de Tradução

### 5.1 Estrutura de diretórios

Crie a seguinte estrutura na raiz do projeto (no mesmo nível de `manage.py`):

```
locale/
├── en/
│   └── LC_MESSAGES/
│       ├── django.po   ← arquivo editável com as traduções
│       └── django.mo   ← arquivo compilado (gerado automaticamente)
├── es/
│   └── LC_MESSAGES/
│       ├── django.po
│       └── django.mo
└── pt_BR/              ← só necessário se pt-br NÃO for a língua-fonte
    └── LC_MESSAGES/
        ├── django.po
        └── django.mo
```

Se o idioma padrão do projeto for `pt-br` e as strings no código estiverem em português, **não é necessário criar arquivos de tradução para `pt_BR`** — o Django usará o `msgid` diretamente.

### 5.2 Formato do arquivo `.po`

Cada arquivo `django.po` começa com um cabeçalho e contém pares `msgid`/`msgstr`:

```po
# Translations for: English
# Project: My Project
msgid ""
msgstr ""
"Project-Id-Version: my-project\n"
"POT-Creation-Date: 2026-01-01 00:00+0000\n"
"PO-Revision-Date: 2026-01-01 00:00+0000\n"
"Language: en\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"
"Plural-Forms: nplurals=2; plural=(n != 1);\n"

# String simples
msgid "Catálogo"
msgstr "Catalog"

# String com variável
msgid "Pedido #%(pk)s"
msgstr "Order #%(pk)s"

# String com plural
msgid "%(count)s produto encontrado"
msgid_plural "%(count)s produtos encontrados"
msgstr[0] "%(count)s product found"
msgstr[1] "%(count)s products found"
```

**Plural-Forms por idioma:**

| Idioma | Plural-Forms |
|---|---|
| Inglês (`en`) | `nplurals=2; plural=(n != 1);` |
| Espanhol (`es`) | `nplurals=2; plural=(n != 1);` |
| Português (`pt_BR`) | `nplurals=2; plural=(n > 1);` |
| Russo (`ru`) | `nplurals=3; plural=(n%10==1...);` |
| Árabe (`ar`) | `nplurals=6; plural=...;` |

---

## 6. Templates

### 6.1 Carregamento obrigatório no topo de cada template

Todo template que usa strings traduzíveis deve começar com:

```html
{% load i18n %}
```

Se o template também usa `{% static %}`:

```html
{% load static i18n %}
```

### 6.2 Template base (`base.html`)

No template base, carregue o idioma atual e os idiomas disponíveis logo após o `{% load %}`:

```html
{% load static i18n %}
{% get_current_language as CURRENT_LANG %}
{% get_available_languages as AVAILABLE_LANGUAGES %}
<!DOCTYPE html>
<html lang="{{ CURRENT_LANG }}">
```

As variáveis `CURRENT_LANG` e `AVAILABLE_LANGUAGES` ficam disponíveis em todo o template e seus filhos.

### 6.3 Padrões de tradução em templates

**String simples:**
```html
<a href="/catalogo/">{% trans "Catálogo" %}</a>
<p>{% trans "Nenhum resultado encontrado." %}</p>
```

**String com variável (blocktrans):**
```html
<h1>{% blocktrans with name=user.full_name %}Olá, {{ name }}{% endblocktrans %}</h1>

<!-- Variável com formatação -->
<span>{% blocktrans with total=cart.total|floatformat:2 %}Total: R$ {{ total }}{% endblocktrans %}</span>
```

**Plural:**
```html
{% blocktrans count count=items|length %}
    {{ count }} item encontrado
{% plural %}
    {{ count }} itens encontrados
{% endblocktrans %}
```

**Atributos HTML:**
```html
<!-- Dentro de atributos, use {% trans %} com aspas ou atribua a variável primeiro -->
<input placeholder="{% trans 'Buscar...' %}">
<button aria-label="{% trans 'Fechar menu' %}">

<!-- Para strings complexas, atribua antes e use a variável -->
{% trans "Descrição do produto" as t_desc %}
<meta name="description" content="{{ t_desc }}">
```

**String de variável dinâmica:**
```html
<!-- Traduz o conteúdo de uma variável (o msgid deve existir no .po) -->
{% trans instrument.name %}
{% trans product.category_label %}
```

### 6.4 Tags i18n que NÃO precisam de `{% load i18n %}` separado

As variáveis `CURRENT_LANG` e `AVAILABLE_LANGUAGES` carregadas no template base ficam disponíveis nos templates filhos via herança, mas o `{% load i18n %}` precisa ser repetido em cada template filho que usa `{% trans %}`.

---

## 7. Seletor de Idioma no Navbar

Implementação completa de um seletor de idioma com dropdown interativo (usa Alpine.js para estado do dropdown).

### 7.1 HTML do seletor (desktop)

Adicione dentro do header/navbar do template base:

```html
<div class="nav-lang-wrapper" x-data="{ open: false }" @click.outside="open = false">

    <!-- Botão que mostra o idioma atual -->
    <button
        class="nav-lang-btn"
        @click="open = !open"
        :aria-expanded="open"
        aria-haspopup="true"
        type="button"
        aria-label="{% trans 'Escolher idioma' %}"
    >
        <span class="nav-lang-current">
            {% if CURRENT_LANG == "pt-br" %}PT
            {% elif CURRENT_LANG == "en" %}EN
            {% elif CURRENT_LANG == "es" %}ES
            {% else %}{{ CURRENT_LANG|upper }}{% endif %}
        </span>
    </button>

    <!-- Dropdown com os idiomas disponíveis -->
    <div class="nav-lang-dropdown" x-show="open" x-cloak>
        {% for code, name in AVAILABLE_LANGUAGES %}
        <form action="{% url 'set_language' %}" method="post">
            {% csrf_token %}
            <!-- Redireciona de volta para a página atual após trocar o idioma -->
            <input name="next" type="hidden" value="{{ request.get_full_path }}">
            <input name="language" type="hidden" value="{{ code }}">
            <button
                class="nav-lang-option {% if code == CURRENT_LANG %}is-active{% endif %}"
                type="submit"
            >
                <span class="nav-lang-code">
                    {% if code == "pt-br" %}PT
                    {% elif code == "en" %}EN
                    {% elif code == "es" %}ES
                    {% else %}{{ code|upper }}{% endif %}
                </span>
                <span class="nav-lang-name">{{ name }}</span>
            </button>
        </form>
        {% endfor %}
    </div>

</div>
```

### 7.2 HTML do seletor (versão mobile/simplificada)

Para menus mobile onde botões side-by-side são mais adequados:

```html
<div class="nav-mobile-lang" aria-label="{% trans 'Escolher idioma' %}">
    {% for code, name in AVAILABLE_LANGUAGES %}
    <form action="{% url 'set_language' %}" method="post">
        {% csrf_token %}
        <input name="next" type="hidden" value="{{ request.get_full_path }}">
        <input name="language" type="hidden" value="{{ code }}">
        <button
            class="nav-mobile-lang-btn {% if code == CURRENT_LANG %}is-active{% endif %}"
            type="submit"
        >
            {% if code == "pt-br" %}PT
            {% elif code == "en" %}EN
            {% elif code == "es" %}ES
            {% else %}{{ code|upper }}{% endif %}
        </button>
    </form>
    {% endfor %}
</div>
```

### 7.3 CSS necessário

```css
/* Wrapper com posição relativa para o dropdown absoluto */
.nav-lang-wrapper {
    position: relative;
}

/* Botão principal */
.nav-lang-btn {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 6px 10px;
    border-radius: 8px;
    border: 1.5px solid rgba(0, 0, 0, .12);
    background: transparent;
    cursor: pointer;
    font-size: 0.78rem;
    font-weight: 700;
    color: var(--text);
    transition: border-color 140ms;
}
.nav-lang-btn:hover {
    border-color: var(--brand, #0f4952);
}

/* Dropdown */
.nav-lang-dropdown {
    position: absolute;
    top: calc(100% + 6px);
    right: 0;
    min-width: 160px;
    background: #fff;
    border: 1.5px solid rgba(0, 0, 0, .1);
    border-radius: 12px;
    padding: 6px;
    box-shadow: 0 8px 24px rgba(0, 0, 0, .1);
    z-index: 100;
}

/* Opção individual */
.nav-lang-option {
    display: flex;
    align-items: center;
    gap: 8px;
    width: 100%;
    padding: 8px 10px;
    border-radius: 8px;
    border: none;
    background: transparent;
    cursor: pointer;
    font-size: 0.82rem;
    color: var(--text);
    text-align: left;
    transition: background 120ms;
}
.nav-lang-option:hover,
.nav-lang-option.is-active {
    background: rgba(15, 73, 82, .08);
    color: var(--brand, #0f4952);
}
.nav-lang-code {
    font-weight: 700;
    width: 24px;
}

/* Alpine.js: oculta elementos com x-cloak antes do JS carregar */
[x-cloak] { display: none !important; }
```

### 7.4 Dependência: Alpine.js

Inclua Alpine.js no `<head>` do template base para o dropdown funcionar:

```html
<script src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js" defer></script>
```

Ou instale via npm: `npm install alpinejs` e importe no bundle JS.

**Sem Alpine.js**: substitua `x-data`, `x-show` e `@click.outside` por JavaScript vanilla ou remova o dropdown e mostre todos os idiomas inline.

### 7.5 Como o `set_language` funciona

Quando o formulário é submetido via POST para `/i18n/set_language/`:

1. Django valida que o código enviado está em `LANGUAGES`
2. Salva o código em `request.session[LANGUAGE_SESSION_KEY]`
3. Opcionalmente salva em cookie (`LANGUAGE_COOKIE_NAME`)
4. Redireciona para o valor do parâmetro `next` (a página atual)
5. Na próxima requisição, `LocaleMiddleware` lê da sessão e ativa o idioma

---

## 8. Models

### 8.1 Importação correta

Em arquivos de models, use sempre `gettext_lazy` (com o `_lazy`), pois models são carregados na inicialização do Django, antes de qualquer requisição:

```python
from django.utils.translation import gettext_lazy as _
```

### 8.2 Choices traduzíveis

```python
class Order(models.Model):
    STATUS_CHOICES = [
        ("draft",    _("Rascunho")),
        ("pending",  _("Pendente")),
        ("paid",     _("Pago")),
        ("failed",   _("Falhou")),
        ("refunded", _("Reembolsado")),
    ]

    status = models.CharField(
        max_length=20,
        choices=STATUS_CHOICES,
        default="pending",
    )
```

### 8.3 verbose_name e verbose_name_plural

```python
class Product(models.Model):
    title = models.CharField(
        max_length=255,
        verbose_name=_("título"),
    )
    is_published = models.BooleanField(
        default=True,
        verbose_name=_("publicado"),
        help_text=_("Marca o produto como visível no catálogo público."),
    )

    class Meta:
        verbose_name = _("produto")
        verbose_name_plural = _("produtos")
        ordering = ["-created_at"]
```

### 8.4 Properties que retornam strings

```python
class CartItem(models.Model):
    @property
    def display_type(self) -> str:
        if self.subscription_plan_id:
            return _("Assinatura")
        return self.product.get_product_type_display()
```

### 8.5 O que NÃO traduzir em models

- Slugs e códigos (usados em URLs e lógica): `("pix", "Pix")` — o código `"pix"` não é traduzido, apenas o label `"Pix"`
- Valores de choices usados em condicionais no código: `if order.status == "paid":` — `"paid"` é o código, não traduzir

---

## 9. Views

### 9.1 Importação para views

Em views, use `gettext` (sem `_lazy`) quando a tradução ocorre dentro do contexto de uma requisição:

```python
from django.utils.translation import gettext as _
```

### 9.2 Tradução de strings em views

```python
def product_list(request):
    # Strings traduzidas em tempo de requisição (contexto disponível)
    hero_title = _("Curadoria profissional para músicos exigentes")
    hero_text = _(
        "Explore o catálogo completo com partituras, video scores e playbacks."
    )
    context = {
        "hero_title": hero_title,
        "hero_text": hero_text,
    }
    return render(request, "catalog/product_list.html", context)
```

### 9.3 Strings com variáveis em views

```python
# Interpolação simples com %
message = _("Bem-vindo, %(name)s!") % {"name": user.full_name}

# Mensagens do Django messages framework
from django.contrib import messages
messages.success(request, _("Pedido criado com sucesso."))
messages.error(request, _("Forma de pagamento indisponível."))
```

### 9.4 Strings em `gettext_lazy` (para uso fora de views)

Use `gettext_lazy` em:
- Definições de models (choices, verbose_name, help_text)
- Definições de forms (labels, help_text, error_messages)
- Settings (nomes de idiomas no `LANGUAGES`)
- Strings definidas fora do ciclo de requisição

```python
# Em forms.py
from django.utils.translation import gettext_lazy as _

class CheckoutForm(forms.Form):
    payment_method = forms.ChoiceField(
        label=_("Forma de pagamento"),
        choices=PAYMENT_CHOICES,
        error_messages={"required": _("Selecione uma forma de pagamento.")},
    )
```

---

## 10. Formulários

### 10.1 Labels e mensagens de erro

```python
from django import forms
from django.utils.translation import gettext_lazy as _

class ContactForm(forms.Form):
    name = forms.CharField(
        label=_("Nome"),
        max_length=100,
        widget=forms.TextInput(attrs={"placeholder": _("Seu nome completo")}),
    )
    email = forms.EmailField(
        label=_("E-mail"),
        error_messages={
            "required": _("O e-mail é obrigatório."),
            "invalid": _("Informe um e-mail válido."),
        },
    )
    message = forms.CharField(
        label=_("Mensagem"),
        widget=forms.Textarea,
    )
```

### 10.2 Validação com mensagens traduzidas

```python
def clean_email(self):
    email = self.cleaned_data.get("email")
    if User.objects.filter(email=email).exists():
        raise forms.ValidationError(_("Este e-mail já está cadastrado."))
    return email
```

---

## 11. Fluxo de Tradução

### 11.1 Ciclo de trabalho

```
1. Escrever código com strings marcadas: _("texto")
        ↓
2. Extrair strings → gerar/atualizar .po
        ↓
3. Traduzir as strings no .po
        ↓
4. Compilar .po → .mo
        ↓
5. Reiniciar servidor (Django carrega .mo na inicialização)
```

### 11.2 Comandos essenciais

**Extrair todas as strings marcadas e atualizar os arquivos `.po`:**

```bash
# Para todos os idiomas configurados:
python manage.py makemessages --all

# Para idiomas específicos:
python manage.py makemessages -l en -l es

# Incluir extensões não-padrão:
python manage.py makemessages -l en -l es --extension html,txt,py

# Ignorar diretórios desnecessários:
python manage.py makemessages -l en -l es --ignore=".venv" --ignore="node_modules"
```

**Compilar `.po` → `.mo` (obrigatório após traduzir):**

```bash
python manage.py compilemessages
```

**Verificar strings não traduzidas:**

```bash
# Lista strings sem tradução no arquivo .po
grep -n "^msgstr \"\"$" locale/en/LC_MESSAGES/django.po
```

### 11.3 Makefile recomendado

Crie (ou adicione ao) `Makefile` na raiz do projeto:

```makefile
# Extrai strings e atualiza arquivos .po para todos os idiomas
messages:
	python manage.py makemessages --all --ignore=".venv" --ignore="node_modules" --ignore="staticfiles"

# Compila .po → .mo (necessário após editar as traduções)
compile:
	python manage.py compilemessages

# Atalho: extrai + compila em sequência
translate: messages compile

.PHONY: messages compile translate
```

Uso:

```bash
make messages    # atualiza .po
# ... editar os .po com as traduções ...
make compile     # compila para .mo
```

### 11.4 Adicionando um novo idioma

1. Adicione o idioma em `LANGUAGES` no settings:
   ```python
   LANGUAGES = [
       ("pt-br", _("Português (Brasil)")),
       ("en",    _("English")),
       ("es",    _("Español")),
       ("fr",    _("Français")),   # ← novo
   ]
   ```

2. Crie o diretório:
   ```bash
   mkdir -p locale/fr/LC_MESSAGES
   ```

3. Extraia as strings para o novo idioma:
   ```bash
   python manage.py makemessages -l fr
   ```

4. Edite `locale/fr/LC_MESSAGES/django.po` e preencha os `msgstr`.

5. Compile:
   ```bash
   python manage.py compilemessages
   ```

6. O idioma aparecerá automaticamente no seletor (via `{% get_available_languages %}`).

### 11.5 Boas práticas para escrever strings traduzíveis

```python
# ✅ Correto: string completa como msgid
_("Seu pedido foi confirmado.")

# ❌ Errado: concatenar strings traduzidas (quebra o contexto do tradutor)
_("Seu pedido") + " " + _("foi confirmado.")

# ✅ Correto: usar %(variável)s para interpolação
_("Olá, %(name)s! Seu pedido #%(id)s foi confirmado.") % {"name": name, "id": order_id}

# ❌ Errado: usar .format() ou f-strings dentro do _()
_(f"Olá, {name}!")          # o extrator não consegue capturar
_("Olá, {}!".format(name))  # idem

# ✅ Correto: plural via ngettext
from django.utils.translation import ngettext
msg = ngettext(
    "%(count)d item adicionado.",
    "%(count)d itens adicionados.",
    count,
) % {"count": count}

# ✅ Correto: strings longas com parênteses
_(
    "Explore kits completos com partituras, video scores e playbacks. "
    "Tudo organizado para você estudar com mais intenção."
)
```

---

## 12. Checklist de Implementação

Use esta lista para implementar i18n do zero em um projeto Django.

### Settings e infraestrutura

- [ ] Importar `gettext_lazy as _` no settings
- [ ] Definir `LANGUAGE_CODE` com o idioma padrão (língua do código-fonte)
- [ ] Definir `LANGUAGES` com todos os idiomas suportados
- [ ] Definir `LOCALE_PATHS = [BASE_DIR / "locale"]`
- [ ] Definir `USE_I18N = True`, `USE_L10N = True`, `USE_TZ = True`
- [ ] Posicionar `LocaleMiddleware` após `SessionMiddleware` e antes de `CommonMiddleware`
- [ ] Adicionar `path("i18n/", include("django.conf.urls.i18n"))` no `urls.py`

### Estrutura de arquivos

- [ ] Criar diretório `locale/` na raiz do projeto
- [ ] Criar subdiretórios para cada idioma: `locale/<código>/LC_MESSAGES/`
- [ ] Executar `makemessages` para gerar os arquivos `.po`
- [ ] Preencher as traduções nos arquivos `.po`
- [ ] Executar `compilemessages` para gerar os `.mo`

### Template base

- [ ] Adicionar `{% load static i18n %}` no topo
- [ ] Adicionar `{% get_current_language as CURRENT_LANG %}` e `{% get_available_languages as AVAILABLE_LANGUAGES %}` logo após o `{% load %}`
- [ ] Definir `<html lang="{{ CURRENT_LANG }}">` na tag HTML
- [ ] Implementar o seletor de idioma no navbar (desktop e mobile)
- [ ] Marcar todas as strings do template base com `{% trans %}` ou `{% blocktrans %}`

### Templates de páginas

- [ ] Adicionar `{% load i18n %}` em cada template que usa strings traduzíveis
- [ ] Marcar strings com `{% trans %}`, `{% blocktrans %}`, ou `{% blocktrans count %}` para plurais
- [ ] Verificar strings em atributos HTML (`placeholder`, `aria-label`, `title`, `alt`)
- [ ] Verificar meta tags (`<meta name="description">`, `<title>`)

### Models

- [ ] Importar `gettext_lazy as _` em cada `models.py`
- [ ] Envolver labels de `choices` com `_(...)`
- [ ] Definir `verbose_name` e `verbose_name_plural` na `class Meta`
- [ ] Envolver `help_text` de campos com `_(...)`

### Views

- [ ] Importar `gettext as _` em cada `views.py` que usa strings
- [ ] Envolver strings de contexto com `_(...)`
- [ ] Envolver mensagens do `messages` framework com `_(...)`
- [ ] Usar `%(variável)s` para interpolação (nunca f-strings dentro do `_()`)

### Formulários

- [ ] Importar `gettext_lazy as _` em cada `forms.py`
- [ ] Envolver `label` e `help_text` dos campos com `_(...)`
- [ ] Envolver `error_messages` com `_(...)`
- [ ] Envolver mensagens de `ValidationError` com `_(...)`

### Makefile e fluxo

- [ ] Criar targets `messages` e `compile` no `Makefile`
- [ ] Documentar o fluxo de atualização de traduções para a equipe
- [ ] Verificar que `.mo` está no `.gitignore` ou incluído no repositório (escolha uma convenção)

### Verificação final

- [ ] Testar troca de idioma via seletor no navegador
- [ ] Verificar que o idioma persiste após navegar para outra página
- [ ] Verificar que o idioma padrão é carregado para usuários sem preferência
- [ ] Verificar strings não traduzidas: `grep -n 'msgstr ""' locale/en/LC_MESSAGES/django.po`
- [ ] Verificar que não há concatenação de strings traduzidas (use `%` com dicionário)
- [ ] Testar plurais com valores 0, 1, e 2+

---

## Referência rápida

| O que | Importação | Onde usar |
|---|---|---|
| Tradução em views | `from django.utils.translation import gettext as _` | `views.py`, dentro de funções |
| Tradução em models/forms | `from django.utils.translation import gettext_lazy as _` | `models.py`, `forms.py`, nível de módulo |
| Plural em views | `from django.utils.translation import ngettext` | Dentro de funções |
| Idioma atual | `{% get_current_language as LANG %}` | Templates |
| Lista de idiomas | `{% get_available_languages as LANGS %}` | Templates |
| String em template | `{% trans "texto" %}` | Templates |
| String com variável | `{% blocktrans with var=value %}{{ var }}{% endblocktrans %}` | Templates |
| Plural em template | `{% blocktrans count n=lista\|length %}singular{% plural %}plural{% endblocktrans %}` | Templates |
| Ativar idioma em código | `from django.utils.translation import activate; activate("en")` | Management commands, tasks |
