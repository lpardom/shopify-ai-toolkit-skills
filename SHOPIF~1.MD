# Shopify AI Toolkit — Skills Agregadas

Fuente: https://github.com/Shopify/Shopify-AI-Toolkit/tree/main/skills  
Total: 20 skills

---

## Índice

1. [shopify-admin](#shopify-admin)
2. [shopify-app-store-review](#shopify-app-store-review)
3. [shopify-custom-data](#shopify-custom-data)
4. [shopify-customer](#shopify-customer)
5. [shopify-dev](#shopify-dev)
6. [shopify-functions](#shopify-functions)
7. [shopify-hydrogen](#shopify-hydrogen)
8. [shopify-liquid](#shopify-liquid)
9. [shopify-onboarding-dev](#shopify-onboarding-dev)
10. [shopify-onboarding-merchant](#shopify-onboarding-merchant)
11. [shopify-partner](#shopify-partner)
12. [shopify-payments-apps](#shopify-payments-apps)
13. [shopify-polaris-admin-extensions](#shopify-polaris-admin-extensions)
14. [shopify-polaris-app-home](#shopify-polaris-app-home)
15. [shopify-polaris-checkout-extensions](#shopify-polaris-checkout-extensions)
16. [shopify-polaris-customer-account-extensions](#shopify-polaris-customer-account-extensions)
17. [shopify-pos-ui](#shopify-pos-ui)
18. [shopify-storefront-graphql](#shopify-storefront-graphql)
19. [shopify-use-shopify-cli](#shopify-use-shopify-cli)
20. [ucp](#ucp)

---

## shopify-admin

**Descripción:** Write or explain Admin GraphQL queries and mutations for apps and integrations that extend the Shopify admin. Use when the user wants to understand, design, or generate the operation itself — even before deciding how to run it. Do NOT choose `admin` first for app or extension config validation — use `use-shopify-cli`. Do NOT choose `admin` first to execute Admin GraphQL now via Shopify CLI or for CLI setup/troubleshooting — use `use-shopify-cli`.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.9.1

### Instrucciones

Asistente para escribir queries/mutations GraphQL contra la **Shopify Admin API**. Siempre:

1. Usa `scripts/search_docs.mjs "<query>"` antes de escribir código.
2. Escribe el código con los resultados.
3. Valida con `scripts/validate.mjs --code '...' --model ... --client-name ... --artifact-id ... --revision ...` antes de responder.
4. Si falla: busca el error, corrige, re-valida (máx. 3 reintentos).
5. Retorna código solo después de que pase la validación.

Reglas clave:
- Enlaza siempre la documentación usada.
- Wrap GraphQL en triple backtick con tipo `graphql`.
- No seleccionar más de 5 campos por nivel.
- Usa `shopify-use-shopify-cli` para ejecutar queries con CLI o validar TOMLs.

---

## shopify-app-store-review

**Descripción:** Run a pre-submission compliance check against your Shopify app's codebase. Reviews App Store requirements and surfaces likely issues before you submit for official review.

**Compatibilidad:** Claude Code, Claude Desktop, Cursor  
**Versión:** 1.11.0

### Instrucciones

Revisor de cumplimiento pre-envío contra la App Store de Shopify.

**Flujo:**
1. Obtener lista de requisitos con: `shopify doc fetch --url https://shopify.dev/docs/apps/launch/app-store-review/app-store-ai-self-review-requirements`
2. Evaluar cada requisito contra el codebase y asignar:
   - ✅ **Likely passing**: evidencia positiva de cumplimiento
   - ❌ **Likely failing**: violación clara detectada
   - ⚠️ **Needs review**: ambigüedad, no se puede confirmar solo con código

**Principios:**
- Ante la duda, marcar como ⚠️ (no hacer pasar silenciosamente).
- Resúmenes breves y específicos.
- Grupos con nota "Applies if…" se omiten si la señal no está presente.
- Grupos "Opt-in:" se omiten salvo que el usuario los pida explícitamente.

**Formato de salida:**
```
### Summary
✅ Likely passing: N
❌ Likely failing: N
⚠️ Needs review: N
⏭️ Groups skipped: N

### ⚠️ Requirements that need review
### ❌ Requirements that are likely failing
### Skipped groups
### Resources
```

---

## shopify-custom-data

**Descripción:** MUST be used first when prompts mention Metafields or Metaobjects. Use Metafields and Metaobjects to model and store custom data for your app.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.10.0

### Instrucciones

**SIEMPRE** usar en este orden:

#### 1. Crear definiciones con TOML (99.99% de los casos)

```toml
# shopify.app.toml

[product.metafields.app.care_guide]
type = "single_line_text_field"
name = "Care Guide"
access.admin = "merchant_read_write"

[metaobjects.app.author]
name = "Author"
display_name_field = "name"
access.storefront = "public_read"

[metaobjects.app.author.fields.name]
name = "Author Name"
type = "single_line_text_field"
required = true
```

**NUNCA** usar `metafieldDefinitionCreate` o `metaobjectDefinitionCreate` salvo casos excepcionales (tipos dinámicos en runtime).

#### 2. Escribir valores

```graphql
mutation {
  metafieldsSet(metafields:[{
    ownerId: "gid://shopify/Product/1234",
    key: "example",
    value: "Hello, World!"
  }]) { ... }
}
```

Usar siempre `metaobjectUpsert` para metaobjects.

#### 3. Leer valores

```graphql
query {
  product(id: "gid://shopify/Product/1234") {
    example: metafield(key: "example") { jsonValue }
  }
}
```

**Reglas críticas de identificación:**
- Metafields definidos con `[product.metafields.app.example]` → `namespace: $app`, `key: example`
- **NUNCA** usar `namespace: app` (incorrecto)
- Metaobjects con `[metaobjects.app.author]` → `type: $app:author`

---

## shopify-customer

**Descripción:** The Customer Account API allows customers to access their own data including orders, payment methods, and addresses.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.10.0

### Instrucciones

Asistente para la **Customer Account API** (distinta de la Admin API). Los clientes solo acceden a sus propios datos.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION` antes de escribir código.
2. Escribir código.
3. Validar con `scripts/validate.mjs --code '...' [--version <api-version>] ...`
4. Si falla: buscar, corregir, re-validar (máx. 3 intentos).

**Consideraciones clave:**
- Requiere autenticación del cliente.
- Clientes solo ven sus propios datos (orders, addresses, payment methods).
- Considerar privacidad y protección de datos.
- Para historial de órdenes, incluir estado de fulfillment y devoluciones.

---

## shopify-dev

**Descripción:** Search Shopify developer documentation across all APIs. Use only when no API-specific skill applies.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.10.0

### Instrucciones

Búsqueda general sobre documentación de Shopify. Usar **SOLO cuando ninguna skill API-específica aplica**.

**Flujo:**
1. Log de activación con `scripts/log_skill_use.mjs ...`
2. Buscar con `scripts/search_docs.mjs "<topic>" ...`

Si el usuario pregunta sobre Admin API, Liquid, Checkout Extensions u otra API nombrada, usar la skill correspondiente en su lugar.

---

## shopify-functions

**Descripción:** Shopify Functions allow developers to customize the backend logic that powers parts of Shopify. Available APIs: Discount, Cart and Checkout Validation, Cart Transform, Pickup Point Delivery Option Generator, Delivery Customization, Fulfillment Constraints, Local Pickup Delivery Option Generator, Order Routing Location Rule, Payment Customization.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.10.0

### Instrucciones

**APIs disponibles:**
- Discount (para cualquier tarea de descuento)
- Delivery Customization
- Payment Customization
- Cart Transform
- Cart and Checkout Validation
- Fulfillment Constraints
- Local Pickup Delivery Option Generator
- Pickup Point Delivery Option Generator
- (Deprecadas: Order Discount, Product Discount, Shipping Discount)

**Flujo obligatorio:**
1. Buscar en docs con `scripts/search_docs.mjs "<query>" --version API_VERSION`
2. Elegir la Function API correcta.
3. Generar con CLI: `shopify app generate extension --template <api> --flavor <rust|vanilla-js|typescript> --name=<name>`
4. Validar con `scripts/validate.mjs ...`

**Lenguaje por defecto: Rust** (si el usuario no especifica).

**Convenciones de nomenclatura Rust:**
- Output `FunctionRunResult` → función `run()`, archivo `src/run.rs`
- Output `FunctionFetchResult` → función `fetch()`, archivo `src/fetch.rs`
- Output `CartLinesDiscountsGenerateRunResult` → función `cart_lines_discounts_generate_run()`, archivo `src/cart_lines_discounts_generate_run.rs`

**Reglas importantes:**
- Functions son puras: sin acceso a red, filesystem, random, ni fecha actual.
- Usar `metaobjectUpsert` con `metafield(namespace: "$app", key: "config")` para configuración.
- En Rust: nunca importar `serde`, `chrono` u otros crates externos (solo `shopify_function`).
- `src/main.rs` es OBLIGATORIO en Rust.
- **NUNCA** ejecutar `shopify app deploy`.

---

## shopify-hydrogen

**Descripción:** (Ver SKILL.md en el repositorio — contenido demasiado extenso para incluir en detalle aquí.)

**Compatibilidad:** Requires Node.js

> El contenido de esta skill superó el límite de tokens en la respuesta del servidor. Referirse directamente a: https://raw.githubusercontent.com/Shopify/Shopify-AI-Toolkit/main/skills/shopify-hydrogen/SKILL.md

---

## shopify-liquid

**Descripción:** Liquid is an open-source templating language created by Shopify. Keywords: liquid, theme, shopify-theme, liquid-component, liquid-block, liquid-section, liquid-snippet, liquid-schemas, shopify-theme-schemas.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.10.0

### Instrucciones

Desarrollador de themes Shopify experto. Genera componentes consistentes con la arquitectura de themes.

**Arquitectura de directorios:**
```
assets/    # CSS, JS, imágenes, fuentes
blocks/    # Componentes reutilizables y anidables
config/    # Configuración global del theme
layout/    # Wrappers de página
locales/   # Traducciones
sections/  # Componentes modulares de página completa
snippets/  # Fragmentos Liquid reutilizables
templates/ # Estructura de páginas (JSON o Liquid)
```

**Reglas de código:**
- SIEMPRE Liquid y HTML válidos.
- SIEMPRE JSON schema correcto en `{% schema %}`.
- Snippets y blocks estáticos requieren tag `{% doc %}`.
- Usar `image_tag`/`image_url` (NO `img_tag`/`img_url` deprecados).
- Sin comentarios en el código.
- Sin referencias a assets externos o librerías JS/CSS.
- Texto visible del usuario: siempre `{{ 'key' | t }}`.

**Flujo obligatorio:**
1. Buscar: `scripts/search_docs.mjs "<query>" ...`
2. Escribir código.
3. Validar: `scripts/validate.mjs --filename <name.liquid> --filetype <tipo> --code <content> ...`

**Gotchas de Liquid:**
- Sin paréntesis en condiciones.
- Sin operador ternario — siempre `{% if %}`.
- `for` loops limitados a 50 iteraciones — usar `{% paginate %}` para arrays grandes.
- `render` crea scope aislado — pasar variables como parámetros.

---

## shopify-onboarding-dev

**Descripción:** Get started building on Shopify. Use when a developer asks to build an app, build a theme, create a dev store, set up a partner account, scaffold a project, or get started developing for Shopify. NOT for merchants managing stores.

**Compatibilidad:** Claude Code, Claude Desktop, Cursor  
**Versión:** 1.11.0

### Instrucciones

**Flujo:**

**Step 1 — Detectar entorno:**

| Señal | Cliente |
|-------|---------|
| "Claude Code" | `claude-code` |
| "Cursor" | `cursor` |
| "VSCode" | `vscode` |
| "Gemini CLI" | `gemini-cli` |

**Step 2 — Instalar prerequisitos:**

```bash
npm install -g @shopify/cli@latest
# o Homebrew (macOS):
brew tap shopify/shopify && brew install shopify-cli
```

Instalar AI toolkit plugin según cliente:

| Cliente | Comando |
|---------|---------|
| `claude-code` | `/plugin marketplace add Shopify/shopify-ai-toolkit` |
| `cursor` | `/add-plugin` → buscar "Shopify" |
| `vscode` | Command Palette → Chat: Install Plugin From Source |
| `gemini-cli` | `gemini extensions install https://github.com/Shopify/shopify-ai-toolkit` |

**Step 3 — Post-instalación:** Preguntar qué quiere construir (App o Theme). Derivar a la skill API-específica correcta.

---

## shopify-onboarding-merchant

**Descripción:** Set up and connect a Shopify store from your AI assistant. For store owners — not developers.

**Compatibilidad:** Claude Code, Claude Desktop, Cursor  
**Versión:** 1.11.0

### Instrucciones

Guía para merchants (sin conocimientos técnicos).

**Flujo:**
1. Detectar OS (macOS/linux/windows).
2. Verificar/instalar Shopify CLI (`shopify version` → si falla, `npm install -g @shopify/cli@latest`).
3. Ofrecer: (1) Crear tienda nueva, (2) Conectar tienda existente.

**Autenticación con la tienda:**
```bash
shopify store auth --store {handle}.myshopify.com --scopes {scopes}
```

Scopes por defecto: productos, inventario, órdenes, clientes, descuentos, themes, contenido, reportes.

**Menú post-conexión:**
1. Agregar/gestionar productos
2. Verificar/actualizar inventario
3. Ver y gestionar órdenes
4. Información de clientes
5. Crear descuentos
6. Personalizar apariencia
7. Ver reportes de ventas
8. Importar productos de otra plataforma

**Importación desde otras plataformas:** Soporta Square, WooCommerce, Etsy, Wix, Amazon, eBay, Clover, Lightspeed R/X, Google Merchant Center. Flujo: exportar CSV → validar → preview → ejecutar con `productSet` mutation → configurar inventario.

---

## shopify-partner

**Descripción:** The Partner API lets you programmatically access data about your Partner Dashboard, including your apps, themes, and affiliate referrals.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.11.0

### Instrucciones

Asistente para la **Partner API** (autenticación a nivel de partner, no de merchant).

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION`
2. Escribir código.
3. `scripts/validate.mjs --code '...' [--version <api-version>] ...`
4. Si falla: buscar, corregir, re-validar (máx. 3 intentos).

**Casos de uso:** apps, themes, referidos de afiliados, transacciones y pagos, datos de instalaciones.

---

## shopify-payments-apps

**Descripción:** The Payments Apps API enables payment providers to integrate their payment solutions with Shopify's checkout.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.11.0

### Instrucciones

Asistente para la **Payments Apps API**. Requiere autenticación de proveedor de pagos y cumplimiento PCI.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION`
2. Escribir código.
3. `scripts/validate.mjs --code '...' [--version <api-version>] ...`
4. Si falla: buscar, corregir, re-validar (máx. 3 intentos).

**Consideraciones clave:**
- Cumplimiento PCI.
- Flujo completo de sesión de pago (inicio → completar).
- Manejo de autorización, captura y liquidación.
- 3D Secure y prevención de fraude.
- Notificaciones webhook.

---

## shopify-polaris-admin-extensions

**Descripción:** Add custom actions and blocks from your app at contextually relevant spots throughout the Shopify Admin.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.11.0

### Instrucciones

**SIEMPRE usar CLI para crear nuevas extensiones:**
```bash
shopify app generate extension --template admin_action --name my-admin-action
shopify app generate extension --template admin_block --name my-admin-block
shopify app generate extension --template admin_link --name admin-link-extension
shopify app generate extension --template admin_print --name my-admin-print-extension
```

**Modelo de componentes por versión de API:**
- `2025-07`: usar componentes **React** de `@shopify/ui-extensions-react/admin`
- Todas las demás versiones (`2025-10`, `2026-01`, `2026-04`, `unstable`, etc.): usar **Polaris web components** con tags `s-*`

**APIs de contexto disponibles:** Customer Segment Template, Discount Function Settings, Order Routing Rule, Product Details Configuration, Product Variant Details Configuration, Purchase Options Card, Validation Settings, Action Extension, Block Extension, Print Action, Standard API, Intents, Picker, Resource Picker, Should Render.

**Reglas de atributos web components:**
- Usar camelCase: `alignItems`, `paddingBlock`, NO kebab-case.
- Booleanos (`disabled`, `loading`): shorthand OK.
- String keywords (`padding`, `gap`, `tone`): SIEMPRE string, NUNCA shorthand.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" --version API_VERSION`
2. Escribir código.
3. `scripts/validate.mjs --code '...' --target <extension-target> [--version ...] ...` (--target es OBLIGATORIO)
4. Si falla: buscar, corregir, re-validar.

---

## shopify-polaris-app-home

**Descripción:** Build your app's primary user interface embedded in the Shopify admin. If the prompt just mentions "Polaris" without more context, assume this API.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.11.0

### Instrucciones

UI principal de la app embebida en el admin de Shopify. Usa Polaris web components (`s-*`).

**APIs disponibles:** App, Config, Environment, Resource Fetching, ID Token, Intents, Loading, Modal API, Navigation, Picker, POS, Print, Resource Picker, Reviews, Save Bar, Scanner, Scopes, Share, Support, Toast, User, Web Vitals.

**Imports:**
```ts
import { useAppBridge } from "@shopify/app-bridge-react";
// Los s-* web components no necesitan import
```

**Patrones disponibles:** Account connection, App card, Callout card, Empty state, Footer help, Index table, Interstitial nav, Media card, Metrics card, Resource list, Setup guide.

**Templates:** Details, Homepage, Index, Settings.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' ...`

---

## shopify-polaris-checkout-extensions

**Descripción:** Build custom functionality that merchants can install at defined points in the checkout flow.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.11.0

### Instrucciones

**CLI para crear nueva extensión:**
```bash
shopify app generate extension --template checkout_ui --name my-checkout-ui-extension
```

**Versión API:** 2026-01

**Targets principales:**
- `purchase.checkout.block.render`
- `purchase.thank-you.block.render`
- `purchase.checkout.cart-line-item.render-after`
- `purchase.checkout.shipping-option-list.render-before`
- `purchase.checkout.payment-method-list.render-after`
- (y más — ver skill completa)

**APIs disponibles:** Addresses, Analytics, Attributes, Buyer Identity, Buyer Journey, Cart Instructions, Cart Lines, Cost, Customer Privacy, Delivery, Discounts, Extension, Gift Cards, Localization, Metafields, Note, Order, Payments, Storefront API, Session Token, Settings, Shop, Storage.

**Imports:**
```tsx
import "@shopify/ui-extensions/preact";
import { render } from "preact";
// s-* web components no necesitan import
```

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" --version API_VERSION`
2. Escribir código.
3. `scripts/validate.mjs --code '...' --target <target> [--version ...] ...` (--target OBLIGATORIO)

---

## shopify-polaris-customer-account-extensions

**Descripción:** Build custom functionality that merchants can install at defined points on the Order index, Order status, and Profile pages in customer accounts.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.11.0

### Instrucciones

**CLI para crear nueva extensión:**
```bash
shopify app generate extension --template=customer_account_ui --name=my_customer_account_ui_extension
```

**Versión API:** 2026-01

**Targets principales:**
- `customer-account.order-status.block.render`
- `customer-account.order-index.block.render`
- `customer-account.profile.block.render`
- `customer-account.order.action.render`
- `customer-account.order.page.render`
- `customer-account.page.render`
- (y más)

**APIs disponibles:** Analytics, Authenticated Account, Customer Account API, Customer Privacy, Extension, Intents, Localization, Navigation, Storefront API, Session Token, Settings, Storage, Toast, Version. Order Status API: Addresses, Attributes, Cart Lines, Cost, Discounts, Gift Cards, Metafields, Order, Shop.

**Imports:**
```tsx
import "@shopify/ui-extensions/preact";
import { render } from "preact";
```

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" --version API_VERSION`
2. Escribir código.
3. `scripts/validate.mjs --code '...' --target <target> [--version ...] ...` (--target OBLIGATORIO)

---

## shopify-pos-ui

**Descripción:** Build retail point-of-sale applications using Shopify's POS UI components. Keywords: POS, Retail, smart grid.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.11.0

### Instrucciones

**SIEMPRE usar CLI para crear extensiones:**
```bash
# Nuevos apps:
shopify app init --template=none --name=<app-name>
# Extensiones (templates: pos_action | pos_block | pos_smart_grid):
shopify app generate extension --name="<name>" --template="pos_smart_grid"
```

**Targets disponibles (34 en total):**

| Surface | Targets clave |
|---------|--------------|
| Smart Grid | `pos.home.tile.render`, `pos.home.modal.render` |
| Cart | `pos.cart.line-item-details.action.render`, `pos.cart.line-item-details.action.menu-item.render` |
| Customer | `pos.customer-details.block.render`, `pos.customer-details.action.render`, `pos.customer-details.action.menu-item.render` |
| Order | `pos.order-details.block.render`, `pos.order-details.action.render` |
| Product | `pos.product-details.block.render`, `pos.product-details.action.render` |
| Post-purchase | `pos.purchase.post.block.render`, `pos.purchase.post.action.render` |
| Receipts | `pos.receipt-header.block.render`, `pos.receipt-footer.block.render` |
| Draft Order | `pos.draft-order-details.block.render`, `pos.draft-order-details.action.render` |
| Register | `pos.register-details.block.render`, `pos.register-details.action.render` |
| Return | `pos.return.post.block.render`, `pos.return.post.action.render` |
| Exchange | `pos.exchange.post.block.render`, `pos.exchange.post.action.render` |
| Events | `pos.cart-update.event.observe`, `pos.transaction-complete.event.observe`, etc. |

**Imports:**
```tsx
import "@shopify/ui-extensions/preact";
import { render } from "preact";
// s-* Polaris web components no necesitan import
```

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" --version API_VERSION`
2. Escribir código.
3. `scripts/validate.mjs --code '...' --target <target> [--version ...] ...` (--target OBLIGATORIO)

---

## shopify-storefront-graphql

**Descripción:** Use for custom storefronts requiring direct GraphQL queries/mutations for data fetching and cart operations. NOT for Web Components.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.10.0

### Instrucciones

Asistente para la **Storefront GraphQL API**. Para storefronts personalizados que necesitan control total sobre fetching y rendering.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION`
2. Escribir código.
3. `scripts/validate.mjs --code '...' [--version <api-version>] ...`
4. Si falla: buscar, corregir, re-validar (máx. 3 intentos).

**Principios:**
- Buscar por nombre de operación o recurso, no el prompt completo.
- Preferir la mutation que directamente cumple la acción pedida.
- Incluir solo campos esenciales para minimizar payload (experiencias customer-facing).

---

## shopify-use-shopify-cli

**Descripción:** Choose when the user needs Shopify CLI to run or fix something now: validate app or extension config on disk, run or troubleshoot store workflows, inventory/product changes by handle/SKU/location, or CLI setup/auth/upgrade issues.

**Compatibilidad:** Requires Node.js  
**Versión:** 1.11.0

### Instrucciones

**Instalación/upgrade:**
```bash
npm install -g @shopify/cli@latest
shopify version
shopify upgrade
shopify commands          # descubrir comandos disponibles
shopify help [command]    # ayuda de un comando específico
```

**Validación de configuración de app:**
```bash
shopify app config validate --json
# Con config específica:
shopify app config validate --json --config <name>
```
(Validar `shopify.app.toml`, `shopify.app.<name>.toml`, `shopify.extension.toml` — NO usar `validate_graphql_codeblocks` para esto)

**Ejecución en tienda (store execution):**
```bash
# 1. Autenticar:
shopify store auth --store <handle>.myshopify.com --scopes <scopes>

# 2. Ejecutar query:
shopify store execute --store <handle>.myshopify.com --query '...'

# 3. Mutation (requiere --allow-mutations):
shopify store execute --store <handle>.myshopify.com --query '...' --allow-mutations
```

**Attribution CLI (para comandos ejecutados por el agente):**
```bash
SHOPIFY_CLI_AGENT_INFO="n:<agent>|v:<version>|p:<provider>|m:<model>" \
SHOPIFY_CLI_AGENT_IDS="s:<session>|r:<run>|i:<instance>" \
shopify ...
```

**Reglas:**
- Para operaciones en tienda: siempre `--store` en ambos comandos.
- Si validated operation es mutation: incluir `--allow-mutations`.
- Preferir `--query` inline sobre archivos `.graphql` separados.
- `--query` o `--query-file` siempre presentes en `shopify store execute`.

---

## ucp

**Descripción:** Use when the user wants to use the UCP CLI to find, compare, buy, or track products from online merchants, or to set up and troubleshoot the local UCP profile.

**Compatibilidad:** Requires UCP CLI  
**Versión:** 1.10.0

### Instrucciones

Toolkit para buyers con intención comercial: encontrar, comprar y rastrear productos.

**Decisión de qué hacer:**

| El buyer dice... | Acción |
|-----------------|--------|
| "Find me X" — sin merchant específico | `ucp catalog search` en catálogo global |
| "Buy this from \<merchant>" | `ucp discover --business <url>` primero |
| "Track my order" | `ucp order get <order_id> --business <url>` |

**Setup requerido (antes de operaciones con merchant específico):**
```bash
ucp profile init --name <local-profile-name>
# Para setup/troubleshoot:
ucp doctor
ucp profile init --name <local-profile-name>
ucp doctor
```

**Búsqueda en catálogo global:**
```bash
ucp catalog search --input '{
  "query": "marathon training shoes",
  "context": { "intent": "...", "address_country": "US", "currency": "USD" },
  "filters": { "price": { "max": 15000 }, "available": true },
  "pagination": { "limit": 10 }
}' --view 'result.products[*].{title: title, seller_domain: variants[0].seller.domain, ...}'
```

**Flujo de compra:**
```bash
# 1. Crear cart:
ucp cart create --business https://<seller-domain> --input '{"line_items": [{"item":{"id":"<variant_id>"},"quantity":1}]}'

# 2. Crear checkout desde cart:
ucp checkout create --business https://<seller-domain> --cart-id <cart_id>

# 3. Completar checkout:
ucp checkout complete <checkout_id> --business https://<seller-domain>
```

**Estados de `result.status`:**
- `completed` → orden creada
- `requires_escalation` → buyer debe ir a `result.continue_url`
- `incomplete` → falta info, usar `checkout update`
- `complete_in_progress` → merchant procesando
- `canceled` → reiniciar

**Reglas de presentación:**
- Renderizar `result.totals[]` en el orden provisto por el merchant (no reordenar).
- Los montos son en unidades menores: `15000` = $150.00 USD.
- Mostrar `result.messages[]` según tipo: `info` (SHOULD), `warning/notice` (MUST), `warning/disclosure` (MUST con posición relativa al item).
- Nunca inventar specs, precios, URLs o disponibilidad.

---

*Última actualización: Junio 2026 — Skills versión desde repositorio oficial de Shopify*
