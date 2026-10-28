# Ejercicio: SLOs, error budget y game day sobre foodie-net

Taller práctico de 45–60 min para acompañar la charla de SRE. Basado en
[The Art of SLOs](https://sre.google/resources/practices-and-processes/art-of-slos/) y en el capítulo
[Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) del SRE Workbook, aplicado a la
infraestructura real de `foodie-net` (Cloud Run + Pub/Sub + Firestore, `us-east1`).

> **Estado:** los comandos están escritos a partir del Terraform y el código del repo, pero **no se han
> ejecutado**. Pruébalos una vez en `dev` antes de usarlos en vivo. Usa siempre el entorno `dev`,
> nunca producción.

## Qué infraestructura se usa

| Pieza | Dónde está | Papel en el ejercicio |
|---|---|---|
| Cloud Run `web` (público, max 3 instancias en dev) | `infra/terraform/envs/dev/main.tf` | Servicio al que se le define el SLO |
| Cloud Run `worker` (privado, max 2) | idem | Consumidor de Pub/Sub; `POST /pubsub/notify` |
| Pub/Sub `notify` → `notify-push` → `notify-dlq` / `notify-dlq-hold` | `modules/pubsub-push` | Escenario de falla (5 intentos, backoff 10–600 s, luego dead-letter) |
| Plan SRE | `docs/plan.md` §5.1–5.4 | SLOs, política de error budget y alertas "oficiales" para comparar |

## Objetivos de aprendizaje

1. Distinguir SLI, SLO, error budget y burn rate con un servicio real.
2. Ver que **un SLI de disponibilidad no ve todas las fallas** (el caso de los mensajes en dead-letter).
3. Configurar alertas burn-rate multi-ventana y entender de dónde salen 14.4 / 6 / 1.
4. Ejecutar un mini game day y cerrar con una decisión de política de error budget y un postmortem.

---

## Parte 1: Elegir SLIs (10 min, en grupo, sin consola)

Reparte los journeys de `docs/plan.md` §5.1 (Browse events, Reserve seats, Notifications, Seat integrity).
Para cada uno, pide al grupo:

- ¿Qué experimenta el usuario cuando falla?
- ¿Qué evento "bueno" y qué evento "total" cuentas? (SLI = buenos / total)
- ¿Qué métrica de GCP lo aproxima? (`run.googleapis.com/request_count`, `request_latencies`,
  `pubsub.googleapis.com/subscription/...`)

Pregunta guía: **"Seat integrity = 100%"**: ¿es un SLO de disponibilidad? ¿Por qué el plan lo trata como
corrección y no como presupuesto de error?

Entregable: una tabla journey → SLI → SLO propuesto. Compárala al final con el plan.

## Parte 2: Crear el SLO de `web` (15 min, consola)

Proyecto `dev` (el de `terraform.tfvars`, `project_id`).

1. **Monitoring → Services**: el servicio Cloud Run `web` aparece automáticamente. Si no, *Define service*.
2. **Create SLO** con estos dos SLIs sobre `web`:
   - **Latencia:** 99.5% de las peticiones en menos de 800 ms (el SLI de "Browse events" del plan).
   - **Disponibilidad:** 99.5% de respuestas no-5xx.
3. Periodo: ventana móvil de 30 días (`rolling`), igual que el plan.
4. Anota el error budget que muestra la consola. Con 99.5% es 0.5% de las peticiones; con 99.9% son ≈ 43 min al mes.

Preguntas: ¿por qué 99.5% y no 99.99%? ¿Qué costaría (en `min_instances`, regiones, dinero) subirlo?

## Parte 3: Alertas burn-rate (10 min)

Burn rate = `fracción de budget consumida × periodo / ventana`. Para un periodo de 30 días (720 h):

| Alerta del plan (§5.3) | Cálculo | Burn rate | Acción |
|---|---|---|---|
| 2% del budget en 1 h | 0.02 × 720 / 1 | **14.4** | Page |
| 5% del budget en 6 h | 0.05 × 720 / 6 | **6** | Page |
| 10% del budget en 3 días | 0.10 × 720 / 72 | **1** | Ticket |

En **Alerting → Create policy → SLO burn rate**, crea dos políticas para el SLO de latencia:

- Lookback 1 h, umbral 14.4 → canal de notificación "page".
- Lookback 6 h, umbral 6 → mismo canal.

Para el ejercicio usa un canal de email propio. El Workbook recomienda ventanas cortas adicionales (1/12 de la
ventana larga) para que la alerta se cierre rápido: coméntalo, no hace falta crearlas.

## Parte 4: Game day (15 min)

Avisa al grupo antes de empezar: quien hace de *Incident Commander* lee las alertas, los demás investigan.

### Falla A: mensajes venenosos en Pub/Sub (el SLI que no ve nada)

El worker valida cada mensaje con `NotifyMessage` y responde **400** a los que nunca podrán tener éxito
(`apps/worker/src/pubsub.ts`). Tras 5 intentos van a `notify-dlq`.

```bash
PROJECT=<project_id de dev>
for i in 1 2 3 4 5; do
  gcloud pubsub topics publish notify --project=$PROJECT --message='{"kind":"no.existe"}'
done
```

Observa durante unos minutos:

- ¿Se movió el SLO de disponibilidad de `web`? (No: es otro servicio y los 400 no son 5xx.)
- ¿Y el del `worker`, si le defines uno? (Tampoco: un 400 no cuenta como 5xx.)
- Mira la métrica `pubsub.googleapis.com/subscription/num_undelivered_messages` de `notify-dlq-hold`:

```bash
gcloud pubsub subscriptions pull notify-dlq-hold --project=$PROJECT --limit=5 --auto-ack
```

**Lección:** el servicio estaba "disponible" y aun así se perdían notificaciones. Hace falta un SLI sobre el
resultado (mensajes entregados vs. mensajes en dead-letter) y una alerta de "DLQ > 0", como dice el plan
("expiry/offer processing stalled (Cloud Tasks backlog or DLQ growth)").

Tarea: crea una alerta sobre `num_undelivered_messages` de `notify-dlq-hold` > 0 durante 5 min.

### Falla B: capacidad (consume el error budget de latencia)

```bash
gcloud run services update web --project=$PROJECT --region=us-east1 --max-instances=1 --concurrency=1

# Carga: con hey (brew install hey) contra la URL de web
hey -z 3m -c 50 "$(gcloud run services describe web --project=$PROJECT --region=us-east1 --format='value(status.url)')/"
```

Con una sola instancia y concurrencia 1, las peticiones se encolan o reciben 429, y la latencia supera 800 ms.
Observa el burn rate del SLO de latencia subir y, si se sostiene, disparar la política de 1 h.

**Revierte siempre al terminar** (valores de dev en Terraform: `max_instances = 3`, concurrencia por defecto):

```bash
gcloud run services update web --project=$PROJECT --region=us-east1 --max-instances=3 --concurrency=80
# o simplemente: terraform apply en infra/terraform/envs/dev
```

El `terraform apply` es la vía segura, porque deja el estado igual al código.

> El módulo `cloud-run-service` no fija la concurrencia, así que el valor a restaurar es el default de Cloud Run (80).

## Parte 5: Cierre (10 min)

1. **Política de error budget.** Con el plan (§5.1): > 50% consumido → congelar features no críticas;
   agotado → solo fixes y parches de seguridad. Con el consumo que vieron, ¿qué decidirían?
2. **Postmortem blameless** de 5 líneas para la Falla A: impacto, detección (¿quién/qué lo notó?), causa,
   qué lo habría detectado antes, acción de seguimiento con responsable.
3. **Runbook.** Escriban el paso a paso para "DLQ creciendo" (mirar `notify-dlq-hold`, decidir entre
   descartar o corregir y re-publicar).

## Preguntas para discutir

- ¿Qué SLI elegirías para "no sobrevender asientos" y por qué no es un budget?
- ¿Qué pasa con el budget si `min_instances = 0` y hay arranques en frío? ¿Merece la pena pagar instancias mínimas?
- Con `max_delivery_attempts = 5` y backoff hasta 600 s, ¿cuánto tarda en aparecer algo en la DLQ? ¿Es aceptable para el SLO de notificaciones (2 min)?

## Limpieza

- Revertir los cambios de la Falla B (`terraform apply`).
- Vaciar `notify-dlq-hold` (la retención es de 7 días, `message_retention_duration = 604800s`).
- Borrar o silenciar las políticas de alerta de prueba y el canal de email.
- Revisar la factura: el presupuesto de dev es de 25 USD/mes (`monthly_budget_usd`).

## Referencias

- [Art of SLOs (Google SRE)](https://sre.google/resources/practices-and-processes/art-of-slos/)
- [SRE Classroom](https://sre.google/classroom/)
- [SRE Workbook: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- `foodie-net/docs/plan.md` §5 (SLOs, alertas, incidentes)
