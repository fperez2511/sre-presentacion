# Guía del facilitador: respuestas y paso a paso

Complemento de [Ejercicio_SLO_foodie-net.md](Ejercicio_SLO_foodie-net.md). Para cada parte del ejercicio
tienes: las **respuestas esperadas** a las preguntas y el **paso a paso de implementación** (consola y
Terraform/`gcloud`).

> **Estado:** nada de esto se ha ejecutado contra GCP. Está escrito a partir del Terraform y del código de
> `foodie-net`. Prueba todo en `dev` antes del taller; los puntos más inciertos están marcados con ⚠️.
> `foodie-net` hoy **no tiene recursos de Monitoring en Terraform** (revisado en `infra/terraform`), así que
> los SLOs y alertas se crean desde cero.

## Preparación (antes del taller)

```bash
export PROJECT=<project_id de dev>      # el de infra/terraform/envs/dev/terraform.tfvars
export REGION=us-east1
gcloud config set project $PROJECT

# 1. Confirmar que hay algo desplegado
gcloud run services list --region=$REGION
gcloud pubsub topics list | grep notify

# 2. Habilitar la API de Monitoring si hace falta
gcloud services enable monitoring.googleapis.com

# 3. Generar algo de tráfico base para que haya datos (10-15 min antes)
WEB=$(gcloud run services describe web --region=$REGION --format='value(status.url)')
hey -z 10m -c 2 -q 2 "$WEB/eventos" &      # brew install hey
```

Si `web` no está desplegado: `terraform apply` en `infra/terraform/envs/dev` (ver su README) y desplegar la
imagen. Sin tráfico los SLOs muestran "no data".

---

## Parte 1: Elegir SLIs

### Respuestas esperadas

Una tabla razonable (coincide con `docs/plan.md` §5.1):

| Journey | SLI (buenos / total) | Métrica de GCP | SLO |
|---|---|---|---|
| Browse events | peticiones a `/eventos` no-5xx y < 800 ms / todas | `run.googleapis.com/request_count` (`response_code_class`), `request_latencies` | 99.5% |
| Reserve seats | peticiones de "Reservar" exitosas (o "sin asientos") en < 2 s / todas | ídem, filtrando la ruta ⚠️ ver nota | 99.9% |
| Notifications | emails aceptados por el proveedor en < 2 min / emitidos | log-based metric en el worker (no existe aún) | 99.0% |
| Seat integrity | eventos con confirmados + retenidos ≤ capacidad / eventos | job de conciliación nocturno | 100% |

⚠️ `request_count` de Cloud Run no etiqueta por ruta. Para filtrar "Reservar" hay que usar logs
(log-based metric sobre `httpRequest.requestUrl`) o un servicio aparte. Para el taller basta con SLIs a nivel
de servicio.

**"Seat integrity = 100%": ¿es disponibilidad? ¿Por qué no es un budget?**

- No es disponibilidad: el servicio puede estar al 100% arriba y aun así vender un asiento dos veces. Es una
  propiedad de **corrección de datos**.
- No se "gasta": un error budget asume que cierta cantidad de fallas es tolerable a cambio de velocidad.
  Un oversell tiene consecuencias de dinero y confianza con un invitado concreto, por eso el plan lo trata
  como incidente (SEV2) y no como consumo de presupuesto.
- Se protege **por diseño**, no por monitoreo: transacciones de Firestore para contar asientos, handlers
  idempotentes, pruebas basadas en propiedades y la conciliación nocturna que abre un ticket ante cualquier
  diferencia (plan §5.5). El monitoreo solo detecta lo que el diseño no evitó.

### Paso a paso

Es un ejercicio en grupo, no hay implementación. Tiempo sugerido: 3 min por journey. Si el grupo se atasca,
pregunta "¿qué vería el invitado si esto falla?" antes de hablar de métricas.

---

## Parte 2: Crear el SLO de `web`

### Respuestas esperadas

- **¿Por qué 99.5% y no 99.99%?**
  1. 99.5% deja ≈ 3.6 h/mes de budget; 99.99% deja ≈ 4.3 min/mes. Un despliegue malo o un problema de
     Firestore de 10 minutos lo agota.
  2. El propio plan acepta que una falla regional son **4–8 h de recuperación** (DR Opción A). Una sola
     caída así (≥ 4 h) ya supera el 0.5% mensual: un SLO más alto que eso sería incoherente con el DR que
     el proyecto eligió.
  3. Subir el SLO se paga: multi-región con balanceador y Cloud Armor, `min_instances` > 0, más pruebas y
     on-call. Para un MVP de eventos en Costa Rica, el usuario no percibe la diferencia entre 99.5% y 99.99%.
- **Budget de referencia:** 0.5% de las peticiones (≈ 3.6 h si lo pensamos como tiempo); 99.9% son ≈ 43 min.

### Paso a paso: consola

1. **Monitoring → Services → Define service**. Si Cloud Run `web` no aparece detectado, elige *Custom* y
   nómbralo `foodie-web` (el Terraform de abajo usa ese id).
2. **Create SLO → Request-based**:
   - Latencia: métrica de latencia de Cloud Run, umbral 800 ms, objetivo 99.5%.
   - Disponibilidad: no-5xx / total, objetivo 99.5%.
3. Compliance period: **Rolling, 30 días**.
4. Anota el budget que muestra la consola.

### Paso a paso: Terraform

Pon esto en un archivo nuevo, por ejemplo `infra/terraform/envs/dev/monitoring.tf` (⚠️ no lo he validado
con `terraform validate`):

```hcl
variable "alert_email" {
  type        = string
  description = "Email que recibe las alertas del taller."
}

locals {
  web_run_filter = "resource.type=\"cloud_run_revision\" resource.label.\"service_name\"=\"web\""
}

resource "google_monitoring_notification_channel" "email" {
  project      = var.project_id
  display_name = "SRE taller (email)"
  type         = "email"
  labels       = { email_address = var.alert_email }
}

resource "google_monitoring_custom_service" "web" {
  project      = var.project_id
  service_id   = "foodie-web"
  display_name = "foodie-net web"
}

resource "google_monitoring_slo" "web_latency" {
  project             = var.project_id
  service             = google_monitoring_custom_service.web.service_id
  slo_id              = "web-latency-800ms"
  display_name        = "99.5% de peticiones < 800 ms (30 d)"
  goal                = 0.995
  rolling_period_days = 30

  request_based_sli {
    distribution_cut {
      distribution_filter = "metric.type=\"run.googleapis.com/request_latencies\" ${local.web_run_filter}"
      range {
        max = 800 # ms
      }
    }
  }
}

resource "google_monitoring_slo" "web_availability" {
  project             = var.project_id
  service             = google_monitoring_custom_service.web.service_id
  slo_id              = "web-availability"
  display_name        = "99.5% de respuestas no-5xx (30 d)"
  goal                = 0.995
  rolling_period_days = 30

  request_based_sli {
    good_total_ratio {
      total_service_filter = "metric.type=\"run.googleapis.com/request_count\" ${local.web_run_filter}"
      bad_service_filter   = "metric.type=\"run.googleapis.com/request_count\" ${local.web_run_filter} metric.label.\"response_code_class\"=\"5xx\""
    }
  }
}
```

Aplicar: `terraform apply -var alert_email=tu@correo.com`. El SLO tarda unos minutos en mostrar datos.

---

## Parte 3: Alertas burn-rate

### Respuestas esperadas

- **Fórmula:** `burn rate = fracción de budget × periodo / ventana`. Con 30 días (720 h):
  1 h y 2% → 14.4; 6 h y 5% → 6; 72 h y 10% → 1.
- **Qué significan:** a burn rate 14.4 el budget de 30 días se agota en `720 / 14.4 = 50 h`; a 6, en 5 días;
  a 1, justo al final del mes.
- **Qué tasa de error dispara cada alerta** (burn rate × budget): con SLO 99.5%, 14.4 corresponde a un
  **7.2%** de peticiones fallidas sostenido 1 h; con 99.9% bastaría un 1.44%. El umbral en burn rate es el
  mismo, pero el error que lo dispara depende del SLO.
- **Por qué dos ventanas:** la larga evita falsos positivos por picos cortos; la corta (1/12 de la larga)
  hace que la alerta se cierre rápido cuando el problema ya terminó. Se combinan con AND.
- **Por qué paginar por burn rate y no por "error rate > X":** pagina según el impacto sobre el objetivo, no
  según un número arbitrario; un 2% de errores a las 4 a.m. puede esperar si el budget lo aguanta.

### Paso a paso: consola

**Alerting → Create policy → Select a metric → SLO burn rate**. Elige el SLO de latencia, lookback `1h`,
umbral `14.4`. Repite con lookback `6h` y umbral `6`. Notificación: el canal de email.

### Paso a paso: Terraform

Añade a `monitoring.tf` (la sintaxis `select_slo_burn_rate` es la del filtro de Cloud Monitoring; ⚠️ valida
que el `name` del SLO se resuelva como esperas):

```hcl
resource "google_monitoring_alert_policy" "web_latency_fast_burn" {
  project      = var.project_id
  display_name = "web latencia: burn rate 14.4 (1 h y 5 min)"
  combiner     = "AND"

  conditions {
    display_name = "burn rate > 14.4 en 1 h"
    condition_threshold {
      filter          = "select_slo_burn_rate(\"${google_monitoring_slo.web_latency.name}\", \"3600s\")"
      comparison      = "COMPARISON_GT"
      threshold_value = 14.4
      duration        = "0s"
    }
  }

  conditions {
    display_name = "burn rate > 14.4 en 5 min"
    condition_threshold {
      filter          = "select_slo_burn_rate(\"${google_monitoring_slo.web_latency.name}\", \"300s\")"
      comparison      = "COMPARISON_GT"
      threshold_value = 14.4
      duration        = "0s"
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.id]
}

resource "google_monitoring_alert_policy" "web_latency_slow_burn" {
  project      = var.project_id
  display_name = "web latencia: burn rate 6 (6 h y 30 min)"
  combiner     = "AND"

  conditions {
    display_name = "burn rate > 6 en 6 h"
    condition_threshold {
      filter          = "select_slo_burn_rate(\"${google_monitoring_slo.web_latency.name}\", \"21600s\")"
      comparison      = "COMPARISON_GT"
      threshold_value = 6
      duration        = "0s"
    }
  }

  conditions {
    display_name = "burn rate > 6 en 30 min"
    condition_threshold {
      filter          = "select_slo_burn_rate(\"${google_monitoring_slo.web_latency.name}\", \"1800s\")"
      comparison      = "COMPARISON_GT"
      threshold_value = 6
      duration        = "0s"
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.id]
}
```

Para el SLO de disponibilidad repite el patrón con `google_monitoring_slo.web_availability`. El tercer nivel
del plan (burn rate 1 en 3 días, **ticket**) no se crea en el taller: el límite de lookback de las alertas
SLO es menor que 3 días en la práctica ⚠️, así que normalmente se resuelve con una consulta/ticket semanal.

---

## Parte 4: Game day

Roles: Incident Commander (decide y comunica), Ops (investiga), Scribe (anota la línea de tiempo). Quien
facilita inyecta las fallas **sin avisar el momento exacto**.

### Falla A: mensajes venenosos

**Inyección**

```bash
for i in 1 2 3 4 5; do
  gcloud pubsub topics publish notify --message='{"kind":"no.existe"}'
done
```

**Qué debería pasar**

- El worker valida con `NotifyMessage`, no coincide con ningún `kind` y responde **400**
  (`apps/worker/src/pubsub.ts`). Pub/Sub reintenta con backoff exponencial de 10 s hasta 600 s y, tras
  `max_delivery_attempts = 5`, manda el mensaje a `notify-dlq`, que guarda `notify-dlq-hold` 7 días.
- Estimación: los reintentos ocurren aproximadamente a los 10, 20, 40 y 80 s → el mensaje llega a la DLQ en
  **≈ 2.5 min** (⚠️ aproximado; el backoff exacto lo decide Pub/Sub).
- Los SLOs de `web` **no se mueven**: es otro servicio. Y aunque definas uno para el `worker` con
  "no-5xx", tampoco: un 400 no es un 5xx.

**Respuestas esperadas a la discusión**

- *¿Por qué el SLI de disponibilidad no lo ve?* Mide "el servicio respondió sin error de servidor", no
  "el trabajo se completó". El worker respondió correctamente 400: el rechazo es el comportamiento
  diseñado. Lo roto es el productor o el contenido del mensaje.
- *¿Qué SLI lo vería?* Uno sobre el resultado: mensajes entregados / (entregados + enviados a la DLQ), o
  simplemente el **tamaño de la DLQ**. Es la lección del plan: "page on symptoms … DLQ growth".
- *¿Es aceptable que tarde ≈ 2.5 min en llegar a la DLQ?* Para el taller sí. Pero ojo: con el mismo backoff,
  una falla **transitoria** del worker de más de ~70-80 s empieza a hacer que las notificaciones superen
  los 2 min del SLO de notificaciones, y una falla de varios minutos las manda a la DLQ aunque los mensajes
  fueran válidos. Eso sí se arregla republicando desde la DLQ (ver runbook).

**Implementación de la alerta de DLQ (Terraform)**

```hcl
resource "google_monitoring_alert_policy" "notify_dlq_not_empty" {
  project      = var.project_id
  display_name = "Pub/Sub notify-dlq-hold tiene mensajes"
  combiner     = "OR"

  conditions {
    display_name = "num_undelivered_messages > 0 por 5 min"
    condition_threshold {
      filter = join(" ", [
        "metric.type=\"pubsub.googleapis.com/subscription/num_undelivered_messages\"",
        "resource.type=\"pubsub_subscription\"",
        "resource.label.\"subscription_id\"=\"notify-dlq-hold\"",
      ])
      comparison      = "COMPARISON_GT"
      threshold_value = 0
      duration        = "300s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MAX"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.id]
}
```

Las métricas de Pub/Sub se muestrean cada 60 s y pueden tener algo de retraso: la alerta no saltará de
inmediato. Dilo en voz alta durante el taller.

### Falla B: capacidad

**Inyección (la del ejercicio original)**

```bash
gcloud run services update web --region=$REGION --max-instances=1 --concurrency=1
hey -z 3m -c 50 "$WEB/eventos"
```

⚠️ **Es la menos predecible.** Con una instancia y concurrencia 1, Cloud Run encola las peticiones unos
segundos y después devuelve **429**. Los 429 son 4xx: no cuentan como "malos" en el SLI no-5xx, y las
rechazadas pueden responder rápido. Puede que el SLO de latencia se mueva y el de disponibilidad no.
**Pruébala antes**; si no se ve nada claro, usa la alternativa:

**Alternativa B′: provocar 5xx de verdad (⚠️ también por probar)**

`web` lee Firestore usando `FIRESTORE_DATABASE_ID`. Apuntarlo a una base inexistente debería hacer fallar
las páginas que leen datos, como `/eventos`:

```bash
gcloud run services update web --region=$REGION \
  --update-env-vars=FIRESTORE_DATABASE_ID=no-existe
hey -z 3m -c 5 -q 5 "$WEB/eventos"
```

Esto genera una revisión nueva. Si la app captura el error y sirve una página vacía con 200, no habrá 5xx;
verifícalo con un `curl -i "$WEB/eventos"` antes del taller.

**Reversión (siempre)**

```bash
cd foodie-net/infra/terraform/envs/dev && terraform apply
```

`terraform apply` es lo más seguro porque deja el servicio igual que el código. Ojo: como `gcloud run
services update` crea revisiones fuera de Terraform, puede que el `apply` solo muestre un diff parcial;
revisa el plan antes de aceptar. La reversión manual de la Falla B es
`--max-instances=3 --concurrency=80` (el módulo no fija la concurrencia; 80 es el default de Cloud Run).

**Qué observar**

- Consola → *Services → SLO → Error budget burn down* y la gráfica de burn rate. A tasa de error del 7.2%,
  la alerta 14.4 de un SLO 99.5% debería dispararse tras cumplirse la ventana corta y la larga ⚠️: con solo
  3 min de carga, la ventana de 1 h tarda en acumular; **esto es esperable**, explícalo: por eso el
  taller no puede esperar una página real, y se enseña la gráfica del burn rate.
- Truco para acelerar en vivo: baja el lookback a 5–10 min y el umbral a 2 en una política de prueba.

---

## Parte 5: Cierre

### Política de error budget: respuesta esperada

Con la política del plan (§5.1):

| Budget consumido | Decisión |
|---|---|
| < 50% | Se sigue desplegando normal. |
| > 50% | Se congelan features no críticas **en ese servicio**; solo cambios de fiabilidad y fixes. |
| 100% | Solo fixes y parches de seguridad hasta que se recupere. |

En el taller, el budget consumido por 3 minutos de falla en una ventana de 30 días es muy pequeño. Lo
importante es que el grupo **justifique** la decisión con números: "consumimos X% del budget en Y min; a este
ritmo se agota en Z h". Buen momento para señalar que la política se acuerda **antes** del incidente, con el
dueño del producto, y no durante.

### Postmortem blameless de 5 líneas (ejemplo para la Falla A)

1. **Impacto:** 5 notificaciones no enviadas entre las HH:MM y HH:MM; ningún usuario real afectado (taller).
2. **Detección:** alerta `notify-dlq-hold > 0` a los ≈ 7 min; los SLOs de `web` no la vieron.
3. **Causa:** se publicaron mensajes con un `kind` inexistente; el worker los rechaza con 400 por diseño.
4. **Qué lo habría detectado antes:** validar el esquema en el productor (el paquete `@foodie/core` ya define
   `Notification`), y un SLI sobre mensajes entregados vs. DLQ.
5. **Acciones:** (a) alerta de DLQ en Terraform — responsable: …, fecha: …; (b) runbook de DLQ —
   responsable: …; (c) tests de contrato entre `web` y `worker`.

Criterio: **sin culpables**, se habla de sistemas ("el productor no validaba"), no de personas.

### Runbook "DLQ creciendo" (borrador)

1. Confirmar: `gcloud pubsub subscriptions describe notify-dlq-hold` y la gráfica de `num_undelivered_messages`.
2. Leer sin borrar: `gcloud pubsub subscriptions pull notify-dlq-hold --limit=10` (**sin** `--auto-ack`).
3. Clasificar: ¿mensajes **inválidos** (esquema/`kind` desconocido) o **válidos** que fallaron por una caída?
4. Inválidos: abrir ticket al equipo del productor, anotar `messageId` y atributos, y solo entonces
   descartar con `--auto-ack`.
5. Válidos: arreglar la causa y republicar al topic `notify`; los handlers son idempotentes, así que
   reprocesar es seguro.
6. Cerrar: la alerta se apaga cuando la DLQ queda vacía; documentar en el postmortem si es SEV2/SEV3.

---

## Preguntas de discusión: respuestas

1. **¿Qué SLI para "no sobrevender asientos" y por qué no es un budget?** Binario por evento:
   `confirmados + retenidos ≤ capacidad`, evaluado por la conciliación nocturna y tras cada escritura crítica.
   Objetivo 100% y cualquier violación es incidente. No es budget por lo explicado en la Parte 1.
2. **Arranques en frío y `min_instances`:** con `min_instances = 0` (default en dev; en prod el comentario del
   Terraform sugiere 1 para evitar cold starts) la primera petición tras un periodo inactivo paga el arranque
   y empeora la cola larga de latencia (p99), que es justo lo que mide el SLO de 800 ms. `startup_cpu_boost`
   ya lo mitiga. Mantener 1 instancia cuesta unos pocos USD al mes con `cpu_idle` activado (⚠️ verifica en la
   calculadora de precios de GCP): suele merecer la pena en producción y no en dev. Es una decisión que se
   toma **mirando el budget de latencia**, no por intuición.
3. **`max_delivery_attempts = 5` y backoff hasta 600 s:** ver la Falla A. Resumen: ≈ 2.5 min hasta la DLQ
   para fallas permanentes; el SLO de notificaciones (2 min) se rompe antes que la DLQ cuando la falla es
   transitoria. Opciones: subir los intentos y bajar el `maximum_backoff`, o aceptar que el SLO de
   notificaciones cubre el caso normal y que la DLQ es la red de seguridad.

---

## Lista de verificación antes del taller

- [ ] `web` y `worker` desplegados en `dev`; `curl "$WEB/healthz"` responde.
- [ ] Tráfico base corriendo desde hace al menos 15 min.
- [ ] SLOs y alertas aplicados (`terraform apply`) y con datos en la consola.
- [ ] Falla A probada: el mensaje llega a `notify-dlq-hold` (`pull` sin `--auto-ack`).
- [ ] Falla B o B′ probada y revertida; confirmado cuál produce señal visible.
- [ ] Canal de email recibe una alerta de prueba.
- [ ] Presupuesto de dev (25 USD/mes) sin consumir de más.
- [ ] Limpieza lista: revertir fallas, vaciar `notify-dlq-hold`, borrar alertas y canal de prueba.
