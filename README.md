# K2 · Landing Minería

Landing de captación B2B para el rubro **minería**. Recibe tráfico de pauta de Meta y envía los
leads al CRM K2 LeadFlow.

| | |
|---|---|
| **Empresa** | K2 Seguridad y Resguardo |
| **Producto** | Landing de captación |
| **Rubro** | Minería |
| **En vivo** | https://mineria.k2.com.pe |
| **Servidor** | EC2 `k2-leads-app` (`i-045a1579dad9fc01d`, 3.88.80.218) |
| **Ruta** | `/var/www/landing` |
| **vhost** | `/etc/nginx/sites-enabled/landing` |

---

## Contenido

```
index.html      Landing (HTML/CSS/JS en un solo archivo, sin build)
gracias.html    Página de conversión — acá se dispara el Lead de Meta
b/index.html    Remanente de un test A/B viejo, SIN USO (ver más abajo)
sucamec-logo.jpg
```

No hay build ni dependencias: se edita el HTML y se copia al servidor.

---

## Medición

| Herramienta | ID | Dónde |
|---|---|---|
| GA4 | `G-JBNFWR6V2T` | landing + gracias |
| Meta Pixel | `1290749182534707` | landing + gracias |
| Clarity | `xbmmc58lbo` | landing + gracias |
| LinkedIn Insight | `9091626` | landing + gracias (sin conversión configurada) |

**Propiedad GA4:** `K2 Landings Pauta B2B (Minería + Agro)` — `535465340`, stream `14771431341`.

### Eventos que emite

| Evento | Cuándo | Parámetros |
|---|---|---|
| `cta_click` | Clic en cualquier CTA que lleva al formulario | `cta_id`, `cta_text`, `cta_location` |
| `form_start` | Primer foco en un campo (una sola vez) | `form_id` |
| `form_field_complete` | Campo válido al salir de él | `field`, `form_id` |
| `form_submit_attempt` | Al enviar, válido o no | `form_id`, `valid` |
| `generate_lead` | Envío exitoso | `value`, `currency`, `lead_id`, `utm_*`, `sector`, `page_variant` |
| `scroll` | 25 / 50 / 75 / 90 % | `percent_scrolled` |
| `section_view` | Sección visible al 30 % | `section_id` |
| `back_to_site_click` | Clic a k2.com.pe desde gracias | `from` |
| `Lead` (Meta) | En `gracias.html` | `value`, `currency`, `content_name`, `eventID` |

**Valores de `cta_id`:** `nav`, `hero`, `servicio_vigilancia`, `servicio_resguardo`,
`servicio_especiales`, `servicio_consultoria`, `servicio_monitoreo`, `cta_final`, `sticky_movil`.

---

## Reglas para no romper el tagueo

Esta landing estuvo con **todo el tagueo de clics muerto** hasta el 11-ago-2026. Las reglas
existen por eso:

1. **El bloque `gtag` va en el `<head>`, siempre.** El bloque `// ====== GA4 TRACKING ======`
   del final arranca con `if(typeof gtag !== 'function') return;`. Los `<script>` clásicos no
   comparten hoisting: si `gtag` se declara en un script posterior, ese guardia corta y **el
   tagueo de clics, scroll y formulario deja de registrarse sin ningún error visible**.
2. **Todo CTA nuevo lleva `data-cta="<identificador>"`.** El evento manda ese valor como
   `cta_id`. No inferir el CTA por su texto: se rompe al cambiar el copy.
3. **El `Lead` de Meta no depende del CRM.** La landing genera siempre un id de evento; si el
   API falla usa uno propio y manda `crm=0`. Antes el `Lead` era condicional y se perdía cada
   vez que el CRM fallaba, mientras GA4 sí contaba el lead.
4. **Verificar con la caché desactivada** y confirmar en GA4 DebugView. No aceptar un "ya está"
   sin captura de Network.

---

## Formulario

10 campos, **7 obligatorios**: intención, nombre, teléfono, RUC, email, empresa, consentimiento.
`puesto` es opcional. `sector` va oculto y fijo en "Minería".

Envía a `POST /crm-intake`, que debe responder `{ ok: true, id: "K2-NNN" }`. Si falla, la landing
igual redirige a `gracias.html` y manda `crm=0` para que se pueda medir cuántos envíos no
llegaron al CRM.

---

## Pendientes conocidos

- **Fricción del formulario:** 7 campos obligatorios para un lead pagado en frío es mucho.
  Exige teléfono **y** email a la vez, y el RUC es obligatorio. Es la fuga medida más grande.
- **`page_variant` siempre vale "A":** el código lee un `input[name="variant"]` que no existe en
  el DOM. Cualquier test A/B en esta landing es inmedible hasta que se agregue ese campo.
- **`b/index.html` es basura:** remanente del 13-jul con copy viejo. Minería no tiene split A/B
  (el split por cookie `k2ab` es solo de agro). Esa carpeta es accesible en
  `mineria.k2.com.pe/b/`, no está medida y puede recibir tráfico de links viejos. Conviene borrarla.
- **LinkedIn sin conversión:** el Insight Tag carga pero nunca se llama `lintrk('track')`.
- **nginx sirve la landing con HTTP 200 en cualquier URL inventada** (catch-all). Eso genera
  `page_view` fantasma. Corregir con `try_files $uri $uri/ =404;`.

---

## Desplegar

```bash
ssh-keygen -t rsa -b 2048 -f /tmp/k2key -N ""
aws ec2-instance-connect send-ssh-public-key \
  --instance-id i-045a1579dad9fc01d --availability-zone us-east-1d \
  --instance-os-user ubuntu --ssh-public-key file:///tmp/k2key.pub

scp -i /tmp/k2key index.html ubuntu@3.88.80.218:/tmp/
ssh -i /tmp/k2key ubuntu@3.88.80.218 \
  'cd /var/www/landing && sudo cp -p index.html index.html.PRE-<CAMBIO>-$(date +%Y%m%d) && sudo cp /tmp/index.html index.html'
```

La instancia **no tiene agente SSM**, por eso el acceso es por EC2 Instance Connect (llave
temporal de 60 segundos). No hay CI/CD.

**Convención:** antes de sobrescribir, backup en el servidor como
`index.html.PRE-<CAMBIO>-<AAAAMMDD>`. Esos backups viven en el servidor, no en el repo.

---

## Backend: a dónde va el lead

`POST /crm-intake` → nginx lo proxea a **Hermes** (`hermes.imperiumtech.ai/api/intake/<id>`),
con la API key en un header del lado del servidor. Responde `{ ok: true, id: "K2-NNN" }`.

> **Hermes es el único CRM.** `crm.k2.com.pe` (el LeadFlow local) está **de baja**: dejó de
> recibir leads de landing el 17-jul-2026, cuando se configuró el proxy. No usar su base para
> nada analítico — quedó congelada y no refleja el volumen real.

### ⚠️ Pendiente antes de apagar el CRM local

La captura de abandono de este formulario (`navigator.sendBeacon('/api/form-sessions')`) pega
al app local en el puerto 3001, **no a Hermes**. Cuando esa instancia se apague, el beacon va a
fallar **en silencio**: la landing no muestra error y tampoco emite evento a GA4.

Hay 449 registros de abandono acumulados ahí (nombre, teléfono, RUC, empresa, UTMs) que se
pierden si no se migran. Al migrar, apuntar el beacon a Hermes y agregar un evento
`form_abandon` a GA4 para que la caída sea visible.
