# Django Internationalization — Implementation Spec

A complete technical and generic guide for implementing multi-language support in Django projects. Covers configuration, templates, models, views, UI language switching, and the translation workflow. Designed to be executed by an AI agent as an implementation specification.

---

## Index

1. [Architectural Decisions](#1-architectural-decisions)
2. [Settings](#2-settings)
3. [Middleware](#3-middleware)
4. [URLs](#4-urls)
5. [Translation File Structure](#5-translation-file-structure)
6. [Templates](#6-templates)
7. [Navbar Language Selector](#7-navbar-language-selector)
8. [Models](#8-models)
9. [Views](#9-views)
10. [Forms](#10-forms)
11. [Translation Workflow](#11-translation-workflow)
12. [Implementation Checklist](#12-implementation-checklist)

---

## 1. Architectural Decisions

Before implementing, define these three choices. They affect all subsequent steps.

### 1.1 Language Detection Strategy

**Option A — Session/Cookie (no URL prefix)**
The language is saved in the user's session. URLs remain the same for all languages (`/catalog/` works in any language).

- **Pros**: Clean URLs, no redirects, simple implementation.
- **Cons**: Links are not shareable with a fixed language; search bots only index the default language.
- **When to use**: SaaS applications, platforms with mandatory login, when multilingual SEO is not critical.

**Option B — URL Prefix (`i18n_patterns`)**
Each language has its own URL prefix (`/en/catalog/`, `/es/catalogo/`).

- **Pros**: Shareable language links; per-language SEO; `hreflang` works correctly.
- **Cons**: Requires redirects when switching languages; internal links must use `{% url %}` with `LocalePrefixPattern`.
- **When to use**: Public websites, e-commerce with multilingual SEO, static content.

**This spec describes Option A (session/cookie)**, which is the most straightforward to implement. Differences for Option B are indicated in notes throughout the document.

### 1.2 Default Language (Source Language)

Define which language will be used to write the source code (strings in `.py` files and templates). Strings remain as they are for this language; `.po` files are created only for others.

Example: If the project is in Portuguese (`pt-br`), the strings in the code are in Portuguese and translation files are created for `en` and `es`.

### 1.3 Supported Languages

List the languages before starting. Each language needs:
- A `locale/<code_with_underscore>/LC_MESSAGES/` directory.
- A translated `django.po` file.
- A compiled `django.mo` file.

---

## 2. Settings

Edit the main project settings file (`config/settings/base.py` or equivalent).

### 2.1 Necessary Import

```python
from django.utils.translation import gettext_lazy as _
```

This import must be at the top of the settings file so that the language names in `LANGUAGES` are translatable.

### 2.2 Required Configurations

```python
# Default application language (language in which the code is written)
LANGUAGE_CODE = "en-us"   # replace as per your project: "pt-br", "es", etc.

# List of supported languages
# The first element is the language code (used in URLs and sessions)
# The second is the display name (wrapped in _() to be translatable)
LANGUAGES = [
    ("en",    _("English")),
    ("pt-br", _("Portuguese (Brazil)")),
    ("es",    _("Spanish")),
]

# Root directory where translation folders reside
# Should be a single directory at the project root
LOCALE_PATHS = [BASE_DIR / "locale"]

# Activates Django's translation system
USE_I18N = True

# Activates localization of dates, numbers, and currencies
USE_L10N = True

# Activates timezone support
USE_TZ = True

# Default server timezone
TIME_ZONE = "UTC"   # adjust to your timezone
```

### 2.3 Notes on Language Codes

Django accepts two language code formats:
- With hyphen: `"pt-br"`, `"zh-hans"` — used in `LANGUAGES` and the session.
- With underscore: `pt_BR`, `zh_Hans` — used in directory names within `locale/`.

Django automatically converts between the two formats. However, to avoid confusion, adopt a consistent convention for directory names. Recommendation:

| Code in `LANGUAGES` | `locale/` Directory |
|---|---|
| `"pt-br"` | `locale/pt_BR/` |
| `"en"` | `locale/en/` |
| `"es"` | `locale/es/` |
| `"zh-hans"` | `locale/zh_Hans/` |

---

## 3. Middleware

### 3.1 Mandatory Position

`LocaleMiddleware` must be **after** `SessionMiddleware` and **before** `CommonMiddleware`. This order is critical: the locale middleware needs the session to read the saved language, and `CommonMiddleware` needs the language to be already active for correct redirects.

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",        # if using WhiteNoise
    "django.contrib.sessions.middleware.SessionMiddleware",   # ← session first
    "corsheaders.middleware.CorsMiddleware",             # if using CORS
    "django.middleware.locale.LocaleMiddleware",         # ← locale HERE
    "django.middleware.common.CommonMiddleware",         # ← common after
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    # other third-party middlewares...
]
```

### 3.2 How LocaleMiddleware Detects Language

The middleware attempts to detect the user's language in this priority order:

1. `request.session[LANGUAGE_SESSION_KEY]` — language saved in the session by `set_language`.
2. Cookie `django_language` — language saved in a cookie.
3. HTTP `Accept-Language` header from the browser.
4. `LANGUAGE_CODE` from settings — default language (final fallback).

---

## 4. URLs

### 4.1 Register i18n URLs (Option A — no prefix)

Add a line to the main URLs file (`config/urls.py`):

```python
from django.urls import include, path

urlpatterns = [
    path("i18n/", include("django.conf.urls.i18n")),  # ← mandatory
    # ... other project URLs
]
```

This exposes the `set_language` view at the `POST /i18n/set_language/` endpoint, which is called by the language switcher form.

### 4.2 Option B — With URL Prefix

If opting for prefixed URLs, replace `urlpatterns` with:

```python
from django.conf.urls.i18n import i18n_patterns
from django.urls import include, path

urlpatterns = [
    path("i18n/", include("django.conf.urls.i18n")),
]

urlpatterns += i18n_patterns(
    path("catalog/", include("apps.catalog.urls")),
    path("library/", include("apps.library.urls")),
    # ... other URLs that should have a language prefix
    prefix_default_language=False,  # /catalog/ instead of /en/catalog/
)
```

With `prefix_default_language=False`, the default language has no URL prefix, while others do (`/pt-br/catalogo/`, `/es/catalogo/`).

---

## 5. Translation File Structure

### 5.1 Directory Structure

Create the following structure at the project root (same level as `manage.py`):

```
locale/
├── en/
│   └── LC_MESSAGES/
│       ├── django.po   ← editable translation file
│       └── django.mo   ← compiled file (auto-generated)
├── es/
│   └── LC_MESSAGES/
│       ├── django.po
│       └── django.mo
└── pt_BR/              ← only needed if English is NOT the source language
    └── LC_MESSAGES/
        ├── django.po
        └── django.mo
```

If the project's default language is `en` and the strings in the code are in English, **you don't need to create translation files for `en`** — Django will use the `msgid` directly.

### 5.2 `.po` File Format

Each `django.po` file starts with a header and contains `msgid`/`msgstr` pairs:

```po
# Translations for: Portuguese (Brazil)
# Project: My Project
msgid ""
msgstr ""
"Project-Id-Version: my-project\n"
"POT-Creation-Date: 2026-01-01 00:00+0000\n"
"PO-Revision-Date: 2026-01-01 00:00+0000\n"
"Language: pt_BR\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"
"Plural-Forms: nplurals=2; plural=(n > 1);\n"

# Simple string
msgid "Catalog"
msgstr "Catálogo"

# String with variable
msgid "Order #%(pk)s"
msgstr "Pedido #%(pk)s"

# String with plural
msgid "%(count)s product found"
msgid_plural "%(count)s products found"
msgstr[0] "%(count)s produto encontrado"
msgstr[1] "%(count)s produtos encontrados"
```

**Plural-Forms by language:**

| Language | Plural-Forms |
|---|---|
| English (`en`) | `nplurals=2; plural=(n != 1);` |
| Spanish (`es`) | `nplurals=2; plural=(n != 1);` |
| Portuguese (`pt_BR`) | `nplurals=2; plural=(n > 1);` |
| Russian (`ru`) | `nplurals=3; plural=(n%10==1...);` |

---

## 6. Templates

### 6.1 Mandatory Load at Top

Every template using translatable strings must start with:

```html
{% load i18n %}
```

If the template also uses `{% static %}`:

```html
{% load static i18n %}
```

### 6.2 Base Template (`base.html`)

In the base template, load the current language and available languages right after the `{% load %}`:

```html
{% load static i18n %}
{% get_current_language as CURRENT_LANG %}
{% get_available_languages as AVAILABLE_LANGUAGES %}
<!DOCTYPE html>
<html lang="{{ CURRENT_LANG }}">
```

The `CURRENT_LANG` and `AVAILABLE_LANGUAGES` variables become available throughout the template and its children.

### 6.3 Translation Patterns in Templates

**Simple string:**
```html
<a href="/catalog/">{% trans "Catalog" %}</a>
<p>{% trans "No results found." %}</p>
```

**String with variable (blocktrans):**
```html
<h1>{% blocktrans with name=user.full_name %}Hello, {{ name }}{% endblocktrans %}</h1>

<!-- Variable with formatting -->
<span>{% blocktrans with total=cart.total|floatformat:2 %}Total: $ {{ total }}{% endblocktrans %}</span>
```

**Plural:**
```html
{% blocktrans count count=items|length %}
    {{ count }} item found
{% plural %}
    {{ count }} items found
{% endblocktrans %}
```

**HTML Attributes:**
```html
<!-- Inside attributes, use {% trans %} with quotes or assign to a variable first -->
<input placeholder="{% trans 'Search...' %}">
<button aria-label="{% trans 'Close menu' %}">

<!-- For complex strings, assign first and use the variable -->
{% trans "Product description" as t_desc %}
<meta name="description" content="{{ t_desc }}">
```

---

## 7. Navbar Language Selector

Complete implementation of a language selector with an interactive dropdown (uses Alpine.js for dropdown state).

### 7.1 Selector HTML (Desktop)

Add inside the header/navbar of the base template:

```html
<div class="nav-lang-wrapper" x-data="{ open: false }" @click.outside="open = false">

    <!-- Button showing the current language -->
    <button
        class="nav-lang-btn"
        @click="open = !open"
        :aria-expanded="open"
        aria-haspopup="true"
        type="button"
        aria-label="{% trans 'Choose language' %}"
    >
        <span class="nav-lang-current">
            {% if CURRENT_LANG == "pt-br" %}PT
            {% elif CURRENT_LANG == "en" %}EN
            {% elif CURRENT_LANG == "es" %}ES
            {% else %}{{ CURRENT_LANG|upper }}{% endif %}
        </span>
    </button>

    <!-- Dropdown with available languages -->
    <div class="nav-lang-dropdown" x-show="open" x-cloak>
        {% for code, name in AVAILABLE_LANGUAGES %}
        <form action="{% url 'set_language' %}" method="post">
            {% csrf_token %}
            <!-- Redirects back to the current page after switching language -->
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

### 7.2 CSS Requirements

```css
/* Wrapper with relative position for the absolute dropdown */
.nav-lang-wrapper {
    position: relative;
}

/* Main button */
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

/* Alpine.js: hide elements with x-cloak before JS loads */
[x-cloak] { display: none !important; }
```

---

## 8. Models

### 8.1 Correct Import

In models files, always use `gettext_lazy` (with `_lazy`), as models are loaded during Django initialization, before any request:

```python
from django.utils.translation import gettext_lazy as _
```

### 8.2 Translatable Choices

```python
class Order(models.Model):
    STATUS_CHOICES = [
        ("draft",    _("Draft")),
        ("pending",  _("Pending")),
        ("paid",     _("Paid")),
        ("failed",   _("Failed")),
        ("refunded", _("Refunded")),
    ]

    status = models.CharField(
        max_length=20,
        choices=STATUS_CHOICES,
        default="pending",
    )
```

---

## 9. Views

### 9.1 View Import

In views, use `gettext` (without `_lazy`) when translation occurs within a request context:

```python
from django.utils.translation import gettext as _
```

### 9.2 String Translation in Views

```python
def product_list(request):
    hero_title = _("Professional curation for demanding musicians")
    hero_text = _("Explore the full catalog with sheet music, video scores, and playbacks.")
    context = {
        "hero_title": hero_title,
        "hero_text": hero_text,
    }
    return render(request, "catalog/product_list.html", context)
```

---

## 11. Translation Workflow

### 11.1 Lifecycle

```
1. Write code with marked strings: _("text")
        ↓
2. Extract strings → generate/update .po
        ↓
3. Translate strings in .po
        ↓
4. Compile .po → .mo
        ↓
5. Restart server (Django loads .mo on startup)
```

### 11.2 Essential Commands

**Extract all marked strings and update `.po` files:**

```bash
# For all configured languages:
python manage.py makemessages --all

# For specific languages:
python manage.py makemessages -l en -l es

# Include non-standard extensions:
python manage.py makemessages -l en -l es --extension html,txt,py
```

**Compile `.po` → `.mo` (required after translating):**

```bash
python manage.py compilemessages
```

---

## 12. Implementation Checklist

- [ ] Import `gettext_lazy as _` in settings.
- [ ] Define `LANGUAGE_CODE` with the default language.
- [ ] Define `LANGUAGES` with all supported languages.
- [ ] Define `LOCALE_PATHS = [BASE_DIR / "locale"]`.
- [ ] Position `LocaleMiddleware` correctly in `MIDDLEWARE`.
- [ ] Add `path("i18n/", include("django.conf.urls.i18n"))` to `urls.py`.
- [ ] Create `locale/` directory and language subdirectories.
- [ ] Implement language selector in the navbar.
- [ ] Mark all template, model, view, and form strings for translation.
- [ ] Run `makemessages`, translate, and run `compilemessages`.
