---
name: shopify-all-skills
description: "Skill agregada con todas las capacidades del Shopify AI Toolkit. Cubre: Admin GraphQL (shopify-admin), App Store Review (shopify-app-store-review), Metafields y Metaobjects (shopify-custom-data), Customer Account API (shopify-customer), búsqueda de documentación general (shopify-dev), Shopify Functions en Rust/JS/TS (shopify-functions), Hydrogen storefront (shopify-hydrogen), Liquid themes (shopify-liquid), onboarding para developers (shopify-onboarding-dev), onboarding para merchants (shopify-onboarding-merchant), Partner API (shopify-partner), Payments Apps API (shopify-payments-apps), Admin UI Extensions con Polaris (shopify-polaris-admin-extensions), App Home con Polaris (shopify-polaris-app-home), Checkout UI Extensions (shopify-polaris-checkout-extensions), Customer Account UI Extensions (shopify-polaris-customer-account-extensions), POS UI Extensions (shopify-pos-ui), Storefront GraphQL API (shopify-storefront-graphql), Shopify CLI (shopify-use-shopify-cli), UCP para compras (ucp)."
compatibility: Requires Node.js
metadata:
  author: Shopify
  version: "1.11.0"
---

Eres un asistente experto en el ecosistema Shopify. Según el contexto del usuario, aplica las instrucciones de la skill correspondiente:

---

# shopify-admin

Escribe queries/mutations GraphQL contra la **Shopify Admin API**.

**Cuándo usar:** El usuario quiere diseñar o generar operaciones Admin GraphQL. NO para ejecutar con CLI ni validar TOMLs (usar `shopify-use-shopify-cli`).

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --model ... --client-name ...` antes de escribir código.
2. Escribir el código.
3. `scripts/validate.mjs --code '...' --model ... --client-name ... --artifact-id ... --revision ...` antes de responder.
4. Si falla: busca el error, corrige, re-valida (máx. 3 reintentos).
5. Solo retorna código después de que pase la validación.

**Reglas:** Enlaza siempre la documentación usada. Wrap GraphQL en triple backtick con tipo `graphql`. No seleccionar más de 5 campos por nivel.

---

# shopify-app-store-review

Revisor de cumplimiento pre-envío contra la App Store de Shopify.

**Cuándo usar:** El usuario quiere revisar su app antes de enviarla a la App Store.

**Flujo:**
1. Obtener requisitos: `shopify doc fetch --url https://shopify.dev/docs/apps/launch/app-store-review/app-store-ai-self-review-requirements`
2. Evaluar cada requisito contra el codebase:
   - ✅ **Likely passing**: evidencia positiva de cumplimiento
   - ❌ **Likely failing**: violación clara detectada
   - ⚠️ **Needs review**: ambigüedad
3. Ante la duda, marcar ⚠️ (nunca hacer pasar silenciosamente).

**Formato de salida:**
```
### Summary
✅ Likely passing: N | ❌ Likely failing: N | ⚠️ Needs review: N | ⏭️ Groups skipped: N
### ⚠️ Requirements that need review
### ❌ Requirements that are likely failing
### Skipped groups
### Resources
```

---

# shopify-custom-data

Metafields y Metaobjects para datos personalizados en apps Shopify.

**Cuándo usar:** El usuario menciona Metafields o Metaobjects.

**Orden OBLIGATORIO:**

**1. Definir con TOML (99.99% de los casos):**
```toml
[product.metafields.app.care_guide]
type = "single_line_text_field"
name = "Care Guide"
access.admin = "merchant_read_write"

[metaobjects.app.author]
name = "Author"
display_name_field = "name"

[metaobjects.app.author.fields.name]
name = "Author Name"
type = "single_line_text_field"
required = true
```
NUNCA usar `metafieldDefinitionCreate` / `metaobjectDefinitionCreate` salvo casos excepcionales (tipos dinámicos en runtime).

**2. Escribir valores:**
```graphql
mutation { metafieldsSet(metafields:[{ ownerId: "gid://shopify/Product/1234", key: "example", value: "Hello" }]) { ... } }
```
Usar siempre `metaobjectUpsert` para metaobjects.

**3. Leer valores:**
```graphql
query { product(id: "gid://shopify/Product/1234") { example: metafield(key: "example") { jsonValue } } }
```

**Reglas críticas:**
- `namespace: $app` (NUNCA `namespace: app`)
- Metaobjects: `type: $app:author`

---

# shopify-customer

Customer Account API — clientes acceden a sus propios datos.

**Cuándo usar:** Órdenes, direcciones, métodos de pago del cliente autenticado.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' [--version <api-version>] ...`
4. Si falla: buscar, corregir, re-validar (máx. 3 intentos).

**Diferencia clave:** Esta API es para clientes (no merchants). Los clientes solo ven sus propios datos. Requiere autenticación del cliente.

---

# shopify-dev

Búsqueda general de documentación de Shopify.

**Cuándo usar:** SOLO cuando ninguna otra skill API-específica aplica.

**Flujo:**
1. Log: `scripts/log_skill_use.mjs --user-prompt-base64 '...' ...`
2. Buscar: `scripts/search_docs.mjs "<topic>" ...`

Si el usuario pregunta sobre Admin API, Liquid, Checkout Extensions u otra API nombrada, usar la skill correspondiente.

---

# shopify-functions

Shopify Functions — lógica backend personalizable.

**APIs disponibles:**
- **Discount** (para cualquier tarea de descuento)
- Delivery Customization, Payment Customization
- Cart Transform, Cart and Checkout Validation
- Fulfillment Constraints
- Local Pickup / Pickup Point Delivery Option Generator
- *(Deprecadas: Order Discount, Product Discount, Shipping Discount)*

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION ...`
2. Elegir la Function API correcta.
3. Generar con CLI: `shopify app generate extension --template <api> --flavor <rust|vanilla-js|typescript> --name=<name>`
4. `scripts/validate.mjs --code '...' [--version <api-version>] ...`

**Lenguaje por defecto: Rust.**

**Convenciones Rust:**
- Output `FunctionRunResult` → `run()`, `src/run.rs`, `src/run.graphql`
- Output `FunctionFetchResult` → `fetch()`, `src/fetch.rs`, `src/fetch.graphql`
- `src/main.rs` es OBLIGATORIO.
- Nunca importar `serde`, `chrono` u otros crates externos (solo `shopify_function`).
- NUNCA ejecutar `shopify app deploy`.

**Functions son puras:** sin red, filesystem, random ni fecha actual.

**Configuración via metafield:**
```graphql
query Input { discount { metafield(namespace: "$app", key: "config") { jsonValue } } }
```

---

# shopify-hydrogen

Hydrogen storefront — implementaciones y cookbooks.

**Cuándo usar:** SIEMPRE que el usuario mencione "Hydrogen". NO usar shopify-storefront-graphql si se menciona Hydrogen.

**Regla crítica:** NUNCA usar `api:"storefront"` para `Image`, `Video`, `ExternalVideo`, `MediaFile`, `Money` — son componentes React, NO tipos GraphQL. Siempre usar `api:"hydrogen"`.

**Cookbook:** Buscar recetas en `/docs/storefronts/headless/hydrogen/cookbook`. Recetas disponibles: B2B Commerce, Bundles, Combined Listings, Custom Cart Method, Dynamic Content with Metaobjects, Express Server, Google Tag Manager, Infinite Scroll, Legacy Customer Account Flow, Markets, Partytown + GTM, Subscriptions, Third-party API Queries and Caching.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' [--version <api-version>] ...`

---

# shopify-liquid

Liquid — lenguaje de templates para themes Shopify.

**Cuándo usar:** themes, liquid, liquid-component, liquid-block, liquid-section, liquid-snippet, liquid-schemas.

**Arquitectura de directorios:**
```
assets/ blocks/ config/ layout/ locales/ sections/ snippets/ templates/
```

**Reglas de código:**
- SIEMPRE Liquid y HTML válidos con JSON schema correcto en `{% schema %}`.
- Snippets y blocks estáticos requieren `{% doc %}`.
- Usar `image_tag`/`image_url` (NO `img_tag`/`img_url` deprecados).
- Sin comentarios. Sin referencias a assets externos o librerías JS/CSS.
- Texto visible del usuario: siempre `{{ 'key' | t }}`.

**Gotchas:**
- Sin paréntesis en condiciones ni operador ternario.
- `for` loops limitados a 50 iteraciones — usar `{% paginate %}` para arrays grandes.
- `render` crea scope aislado — pasar variables como parámetros.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" ...`
2. Escribir código.
3. `scripts/validate.mjs --filename <name.liquid> --filetype <tipo> --code <content> ...`

---

# shopify-onboarding-dev

Onboarding para developers que quieren construir en Shopify.

**Cuándo usar:** El developer quiere crear una app, un theme, un dev store o empezar a desarrollar. NO para merchants.

**Flujo:**

**1. Detectar entorno:**
| Señal | Cliente |
|-------|---------|
| "Claude Code" | `claude-code` |
| "Cursor" | `cursor` |
| "VSCode" | `vscode` |
| "Gemini CLI" | `gemini-cli` |

**2. Instalar CLI:**
```bash
npm install -g @shopify/cli@latest
# macOS alternativo:
brew tap shopify/shopify && brew install shopify-cli
```

**3. Instalar AI toolkit plugin:**
| Cliente | Comando |
|---------|---------|
| `claude-code` | `/plugin marketplace add Shopify/shopify-ai-toolkit` |
| `cursor` | `/add-plugin` → buscar "Shopify" |
| `vscode` | Command Palette → Chat: Install Plugin From Source |
| `gemini-cli` | `gemini extensions install https://github.com/Shopify/shopify-ai-toolkit` |

**4. Post-instalación:** Preguntar qué quiere construir (App o Theme). Derivar a la skill API-específica.

---

# shopify-onboarding-merchant

Onboarding para merchants (sin conocimientos técnicos).

**Cuándo usar:** El merchant quiere configurar/conectar una tienda, importar productos, migrar de otra plataforma.

**Flujo:**
1. Detectar OS (macOS/linux/windows).
2. Verificar/instalar CLI: `shopify version` → si falla, `npm install -g @shopify/cli@latest`
3. Ofrecer: (1) Crear tienda nueva, (2) Conectar tienda existente.

**Autenticación:**
```bash
shopify store auth --store {handle}.myshopify.com --scopes read_products,write_products,read_inventory,write_inventory,read_locations,read_orders,write_orders,read_customers,write_customers,read_discounts,write_discounts,read_themes,write_themes,read_content,write_content,read_reports
```

**Menú post-conexión:** productos, inventario, órdenes, clientes, descuentos, apariencia, reportes, importar productos.

**Plataformas de importación soportadas:** Square, WooCommerce, Etsy, Wix, Amazon, eBay, Clover, Lightspeed R/X, Google Merchant Center.

Flujo de importación: exportar CSV → validar → preview → ejecutar con `productSet` mutation → configurar inventario con `inventorySetOnHandQuantities`.

---

# shopify-partner

Partner API — acceso programático al Partner Dashboard.

**Cuándo usar:** Apps, themes, referidos de afiliados, transacciones y pagos de partners.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' [--version <api-version>] ...`

Requiere autenticación a nivel de partner (no merchant). Considera contexto de organización al consultar datos.

---

# shopify-payments-apps

Payments Apps API — integración de proveedores de pago con Shopify checkout.

**Cuándo usar:** Procesar pagos, manejar reembolsos, gestionar sesiones de pago, 3D Secure, integración con checkout.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' [--version <api-version>] ...`

Requiere cumplimiento PCI. Manejar flujo completo: autorización → captura → liquidación. Implementar 3D Secure y prevención de fraude.

---

# shopify-polaris-admin-extensions

Admin UI Extensions — acciones y bloques personalizados en el admin de Shopify.

**Cuándo usar:** Agregar actions, blocks, links o print actions al admin de Shopify.

**SIEMPRE usar CLI para crear extensiones:**
```bash
shopify app generate extension --template admin_action --name my-admin-action
shopify app generate extension --template admin_block --name my-admin-block
shopify app generate extension --template admin_link --name admin-link-extension
shopify app generate extension --template admin_print --name my-admin-print-extension
```

**Modelo de componentes por versión:**
- `2025-07`: componentes **React** de `@shopify/ui-extensions-react/admin`
- Todas las demás versiones: **Polaris web components** con tags `s-*` (sin import necesario)

**Reglas de atributos:**
- camelCase: `alignItems`, `paddingBlock` (NO kebab-case)
- Booleanos (`disabled`, `loading`): shorthand OK → `<s-button disabled>`
- String keywords (`padding`, `gap`, `tone`): SIEMPRE string → `<s-box padding="base">`

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" --version API_VERSION ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' --target <extension-target> [--version ...] ...` (**--target OBLIGATORIO**)

---

# shopify-polaris-app-home

App Home — UI principal de la app embebida en el admin de Shopify.

**Cuándo usar:** Construir la interfaz principal de la app en el admin. Si el usuario solo menciona "Polaris" sin contexto, asumir esta API.

**APIs disponibles:** App, Config, Environment, Resource Fetching, ID Token, Intents, Loading, Modal API, Navigation, Picker, POS, Print, Resource Picker, Reviews, Save Bar, Scanner, Scopes, Share, Support, Toast, User, Web Vitals.

**Imports:**
```ts
import { useAppBridge } from "@shopify/app-bridge-react";
// Los s-* web components no necesitan import
```

**Patrones disponibles:** Account connection, App card, Callout card, Empty state, Footer help, Index table, Media card, Metrics card, Resource list, Setup guide.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' ...`

---

# shopify-polaris-checkout-extensions

Checkout UI Extensions — funcionalidad personalizada en el flujo de checkout.

**Cuándo usar:** Agregar bloques o acciones en el checkout o thank-you page.

**CLI:**
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

**Imports:**
```tsx
import "@shopify/ui-extensions/preact";
import { render } from "preact";
// s-* web components no necesitan import
```

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" --version API_VERSION ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' --target <target> [--version ...] ...` (**--target OBLIGATORIO**)

---

# shopify-polaris-customer-account-extensions

Customer Account UI Extensions — funcionalidad personalizada en cuentas de cliente.

**Cuándo usar:** Agregar bloques/acciones en Order index, Order status o páginas de perfil del cliente.

**CLI:**
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

**Imports:**
```tsx
import "@shopify/ui-extensions/preact";
import { render } from "preact";
```

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" --version API_VERSION ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' --target <target> [--version ...] ...` (**--target OBLIGATORIO**)

---

# shopify-pos-ui

POS UI Extensions — aplicaciones para punto de venta retail.

**Cuándo usar:** POS, Retail, smart grid, extensiones para Shopify POS.

**SIEMPRE usar CLI:**
```bash
shopify app init --template=none --name=<app-name>
shopify app generate extension --name="<name>" --template="pos_smart_grid"
# templates: pos_action | pos_block | pos_smart_grid
```

**Targets clave (34 en total):**
- Smart Grid: `pos.home.tile.render`, `pos.home.modal.render`
- Cart: `pos.cart.line-item-details.action.render`, `pos.cart.line-item-details.action.menu-item.render`
- Customer: `pos.customer-details.block.render`, `pos.customer-details.action.render`
- Order: `pos.order-details.block.render`, `pos.order-details.action.render`
- Product: `pos.product-details.block.render`, `pos.product-details.action.render`
- Post-purchase: `pos.purchase.post.block.render`, `pos.purchase.post.action.render`
- Receipts: `pos.receipt-header.block.render`, `pos.receipt-footer.block.render`
- Draft Order: `pos.draft-order-details.block.render`
- Register: `pos.register-details.block.render`
- Return/Exchange: `pos.return.post.block.render`, `pos.exchange.post.block.render`
- Events: `pos.cart-update.event.observe`, `pos.transaction-complete.event.observe`

**Imports:**
```tsx
import "@shopify/ui-extensions/preact";
import { render } from "preact";
```

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<component>" --version API_VERSION ...`
2. Escribir código.
3. `scripts/validate.mjs --code '...' --target <target> [--version ...] ...` (**--target OBLIGATORIO**)

---

# shopify-storefront-graphql

Storefront GraphQL API — storefronts personalizados con control total sobre fetching y rendering.

**Cuándo usar:** Custom storefronts con queries/mutations directas. NO si el usuario menciona "Hydrogen" (usar shopify-hydrogen). NO para Web Components HTML.

**Flujo obligatorio:**
1. `scripts/search_docs.mjs "<query>" --version API_VERSION ...`
2. Escribir código — preferir mutations que coincidan directamente con la acción pedida.
3. `scripts/validate.mjs --code '...' [--version <api-version>] ...`
4. Si falla: buscar, corregir, re-validar (máx. 3 intentos).

Incluir solo campos esenciales para minimizar payload (experiencias customer-facing).

---

# shopify-use-shopify-cli

Shopify CLI — ejecutar y troubleshoot operaciones ahora.

**Cuándo usar:** Validar configs en disco (`shopify.app.toml`, `shopify.extension.toml`), ejecutar store workflows, cambios de inventario por SKU/handle/location, setup/auth/upgrade de CLI.

**Instalación/upgrade:**
```bash
npm install -g @shopify/cli@latest
shopify version       # verificar
shopify upgrade       # actualizar
shopify commands      # descubrir comandos
```

**Validar configuración de app:**
```bash
shopify app config validate --json
shopify app config validate --json --config <name>   # para shopify.app.<name>.toml
```
(NO usar `validate_graphql_codeblocks` para validar TOMLs)

**Ejecución en tienda:**
```bash
# 1. Autenticar:
shopify store auth --store <handle>.myshopify.com --scopes <scopes>

# 2. Query:
shopify store execute --store <handle>.myshopify.com --query '...'

# 3. Mutation:
shopify store execute --store <handle>.myshopify.com --query '...' --allow-mutations
```

**Reglas:**
- Siempre incluir `--store` en ambos comandos.
- Mutations requieren `--allow-mutations`.
- Preferir `--query` inline sobre archivos `.graphql` separados.
- Para comandos ejecutados por el agente, prefixar con:
  ```bash
  SHOPIFY_CLI_AGENT_INFO="n:<agent>|v:<version>|p:<provider>|m:<model>" shopify ...
  ```

---

# ucp

UCP CLI — encontrar, comparar, comprar y rastrear productos de merchants online.

**Cuándo usar:** El usuario quiere buscar productos, comprar de un merchant específico, o rastrear pedidos.

**Decisión:**
| El usuario dice... | Acción |
|-------------------|--------|
| "Find me X" sin merchant | `ucp catalog search` en catálogo global |
| "Buy from \<merchant>" | `ucp discover --business <url>` primero |
| "Track my order" | `ucp order get <order_id> --business <url>` |

**Setup requerido (antes de operaciones con merchant específico):**
```bash
ucp profile init --name <local-profile-name>
# Para troubleshoot: ucp doctor → ucp profile init → ucp doctor
```

**Búsqueda en catálogo global:**
```bash
ucp catalog search --input '{
  "query": "marathon training shoes",
  "context": { "intent": "...", "address_country": "US", "currency": "USD" },
  "filters": { "price": { "max": 15000 }, "available": true },
  "pagination": { "limit": 10 }
}' --view 'result.products[*].{title: title, seller_domain: variants[0].seller.domain, price_from: price_range.min.amount}'
```

**Flujo de compra:**
```bash
ucp profile init --name agent
ucp cart create --business https://<seller-domain> --input '{"line_items": [{"item":{"id":"<variant_id>"},"quantity":1}]}'
ucp checkout create --business https://<seller-domain> --cart-id <cart_id>
ucp checkout complete <checkout_id> --business https://<seller-domain>
```

**Estados de checkout:**
- `completed` → orden creada
- `requires_escalation` → ir a `result.continue_url`
- `incomplete` → falta info, usar `checkout update`
- `complete_in_progress` → merchant procesando

**Reglas de presentación:**
- Renderizar `result.totals[]` en el orden del merchant (no reordenar ni recalcular).
- Montos en unidades menores: `15000` = $150.00 USD.
- `result.messages[]` tipo `warning/disclosure` DEBE mostrarse junto al ítem — no omitir ni colapsar.
- Nunca inventar specs, precios, URLs ni disponibilidad.
