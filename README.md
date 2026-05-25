# F5 iRule — URL-gebaseerde client certificaat autorisatie

Deze iRule maakt het mogelijk om per URL-prefix te bepalen welke combinaties van client certificaat **serienummer** (SN) en **common name** (CN) toegang krijgen. De configuratie gebeurt volledig via data groups, zonder de iRule aan te passen.

## Werking

De iRule werkt met twee lagen van data groups:

- **Hoofd data group** (`url_to_certgroup_dg`) — koppelt een URL-prefix aan een specifieke cert data group.
- **Cert data groups** (bijv. `certs_api_orders`) — koppelen een serienummer aan een toegestane common name.

## Serienummer notatie

F5 geeft serienummers via `X509::serial` terug in de notatie met dubbele punten:

```
1a:2b:3c:4d:5e:6f
```

Gebruik deze notatie exact als sleutel in de cert data groups. `X509::serial` geeft altijd kleine letters terug op BIG-IP — gebruik daarom consequent kleine letters in de data groups. Controleer het exacte formaat door `static::debug` tijdelijk op `1` te zetten en een testverzoek te doen.

## Data group structuur

### url_to_certgroup_dg (type: string)

```
"/api/orders"   := "certs_api_orders",
"/api/invoices" := "certs_api_invoices",
"/admin"        := "certs_admin",
```

### Cert data group, bijv. certs_api_orders (type: string)

```
# Sleutel  = serienummer zoals X509::serial het teruggeeft (kleine letters, dubbele punten)
# Waarde   = verwachte common name  |  * = alleen SN check, CN wordt genegeerd

"1a:2b:3c:4d:5e:6f" := "orders-client-prod",
"0f:1e:2d:3c:4b:5a" := "*",
```

> **Wildcard CN:** Gebruik `*` als value als het serienummer voldoende is en de CN niet gecontroleerd hoeft te worden.

## Logging

Logging is centraal in en uit te zetten via `RULE_INIT`:

```tcl
set static::debug 1   # aan
set static::debug 0   # uit
```

Na het aanpassen hoeft alleen de iRule opnieuw opgeslagen te worden — geen herstart vereist. Logregels worden weggeschreven naar `local0.` en zijn zichtbaar via `/var/log/ltm`.

### Voorbeeld logregels

```
ALLOW | URI: /api/orders/123 | CN: myclient | SN: 1a:2b:3c | Client IP: 1.2.3.4
DENY - Cert mismatch | URI: /api/orders/123 | CN: wrongclient | SN: 1a:2b:3c | Expected CN: myclient | Client IP: 1.2.3.4
DENY - URL not configured | URI: /unknown | CN: myclient | SN: 1a:2b:3c | Client IP: 1.2.3.4
```

## Vereisten

- Virtual Server met een **Client SSL profile** waarbij client certificates op `require` of `request` staat.
- BIG-IP versie 11.x of hoger.

> **Let op:** URL-matching werkt op basis van `starts_with`. Zorg dat prefixes specifiek genoeg zijn om onbedoelde overlap te voorkomen. Elke URL komt slechts één keer voor in de hoofd data group.

## Licentie

MIT
