# Memory Box - n8n

Proyecto n8n para integrar notificaciones con el backend [memory-box-back](https://github.com/your-org/memory-box-back).

## Notificaciones

- **Nuevo pedido** (acción `send_order`): el backend llama al webhook `new-order`. El workflow envía por **WhatsApp** (Twilio) un mensaje al número admin.

- **Pedido finalizado** (cuando en el dashboard se pasa de "En Proceso" a "Finalizado"): el backend llama al webhook `order-finalized`. El workflow envía por **WhatsApp** al **teléfono del cliente** un mensaje con saludo por nombre, aviso de que la cajita está lista y el saldo pendiente (precio del tipo de cajita menos seña).

## Requisitos

- Docker y Docker Compose
- Cuenta [Telegram Bot](https://t.me/BotFather) (gratis)
- Cuenta [Twilio](https://www.twilio.com) con WhatsApp habilitado (opcional, para WhatsApp)

## Inicio rápido

### 1. Clonar y configurar

```bash
cd memory-box-n8n
cp .env.example .env
# Editar .env con tus valores
```

### 2. Variables en .env

| Variable | Descripción |
|----------|-------------|
| `N8N_BASIC_AUTH_PASSWORD` | Contraseña para acceder a n8n |
| `TELEGRAM_BOT_TOKEN` | Token del bot (BotFather) |
| `TELEGRAM_CHAT_ID` | ID del chat o grupo admin |
| `TWILIO_ACCOUNT_SID` | Account SID de Twilio |
| `TWILIO_AUTH_TOKEN` | Auth Token de Twilio |
| `TWILIO_WHATSAPP_FROM` | Número WhatsApp de Twilio (ej: +14155238886) |
| `ADMIN_WHATSAPP_NUMBER` | Un solo admin (ej: +5491112345678) |
| `ADMIN_WHATSAPP_NUMBERS` | Varios admins, separados por coma (ej: +54911a,+54911b). Si está definido, se usa en lugar de `ADMIN_WHATSAPP_NUMBER` |

**Cómo obtener TELEGRAM_CHAT_ID:**

- Envía un mensaje a tu bot
- Visita: `https://api.telegram.org/bot<TOKEN>/getUpdates`
- Busca `"chat":{"id": -123456789}`

### 3. Levantar n8n

```bash
docker compose up -d
```

n8n queda en **http://localhost:5678**

### 4. Importar workflow e iniciar webhook

1. Entra a http://localhost:5678
2. Menú (⋮) → **Import from File** → importa `workflows/new-order-notify.json` y también `workflows/order-finalized-notify.json`.
3. En ambos workflows, configura la credencial del nodo HTTP Request a Twilio: **HTTP Basic Auth** → User = Twilio Account SID, Password = Twilio Auth Token.
4. Conecta si falta: **2_Normalizar** → nodo WhatsApp → **4_Responder**.
5. Activa ambos workflows (Publish).

### 5. URLs de los webhooks

Una vez activos:

| Webhook | Test | Producción |
|---------|------|------------|
| Nuevo pedido | `http://localhost:5678/webhook-test/new-order` | `http://localhost:5678/webhook/new-order` |
| Pedido finalizado | `http://localhost:5678/webhook-test/order-finalized` | `http://localhost:5678/webhook/order-finalized` |

Para producción, usa una URL pública (ej. ngrok o tu dominio).

### 6. Configurar el backend (memory-box-back)

En el `.env` del backend:

```
N8N_WEBHOOK_URL=http://localhost:5678/webhook/new-order
N8N_WEBHOOK_FINALIZED_URL=http://localhost:5678/webhook/order-finalized
```

En producción, sustituye por tus URLs públicas.

## Payloads

**Webhook `new-order`** (nuevo pedido):

```json
{
  "order_id": 123,
  "client_name": "Juan Pérez",
  "phone": "+5491112345678",
  "box_type": "with_light",
  "variant": "graphite_light",
  "shipping_option": "pickup_uber",
  "status": "in_progress",
  "created_at": "2025-02-18T12:00:00Z"
}
```

**Webhook `order-finalized`** (pedido pasado a Finalizado):

```json
{
  "order_id": 123,
  "first_name": "Juan",
  "phone": "+5493513385438939",
  "saldo_pendiente": 22250
}
```

El backend calcula `first_name` (primera parte del nombre del cliente), `saldo_pendiente` = precio total del tipo de cajita − seña (seña = mitad: con luz `(price_con_luz + price_pilas) / 2`, sin luz `price_sin_luz / 2`).

## Solo WhatsApp por ahora

El workflow actual solo envía por WhatsApp (Twilio). Telegram se agregará más adelante.

## Twilio: error "Could not find a Channel with the specified From address" (63007)

Ese error significa que el número **From** no está habilitado para WhatsApp en tu cuenta Twilio. No basta con tener un número de Twilio: tiene que ser el que Twilio asigna para WhatsApp.

1. Entra a [Twilio Console → Messaging → Try it out → Send a WhatsApp message](https://www.twilio.com/console/sms/whatsapp/sandbox) (o [WhatsApp Senders](https://www.twilio.com/console/messaging/whatsapp/senders)).
2. Si usas **Sandbox**: copia el número que aparece como "Sandbox number" (ej. `+14155238886`). Ese es el que debe ir en **From** en el workflow. El admin (+5493513392082) debe haber unido el sandbox (envía el código que te muestra Twilio a ese número por WhatsApp).
3. Si tienes **WhatsApp Business** (número aprobado): usa ese número en **From**.
4. En n8n, en el nodo **3_WhatsApp** → Body Parameters → **From** → pon `whatsapp:+XXXXXXXXXX` con el número exacto que te muestra Twilio para WhatsApp (no el de SMS/llamadas si es distinto).
5. Si cambias el número, actualiza también el workflow JSON (`TWILIO_WHATSAPP_FROM` en `.env` y el valor fijo en `workflows/new-order-notify.json` si lo usas ahí).

## Deploy en mark1 (MicroK8s, mismo namespace que back/front)

n8n corre en el namespace `memory-box-prod` junto al back y el front. El back llama a los webhooks por nombre de servicio: `http://memory-box-n8n:5678/webhook/...`.

1. **Clonar** en mark1 (con deploy key) en `~/workspaces/memory-box/repos/memory-box-n8n`.
2. **Secret con tu .env:** En mark1, desde el repo, crear el secret con los datos reales (copiar tu `.env` al servidor o crear el secret a mano):
   ```bash
   kubectl create secret generic memory-box-n8n-secret -n memory-box-prod \
     --from-env-file=.env --dry-run=client -o yaml | kubectl apply -f -
   ```
3. **Aplicar k8s:**
   ```bash
   microk8s kubectl apply -k k8s/microk8s/overlays/prod -n memory-box-prod
   ```
4. **UI de n8n:** NodePort **30578**. Desde la red local: `http://192.168.88.50:30578`. Entrar (admin / tu contraseña), importar los dos workflows desde `workflows/`, configurar credencial Twilio y activar ambos.
5. **Back:** El ConfigMap del back ya incluye `N8N_WEBHOOK_URL` y `N8N_WEBHOOK_FINALIZED_URL` apuntando a `http://memory-box-n8n:5678/...`. Tras desplegar n8n, reiniciar el back para que tome las variables: `kubectl rollout restart deployment memory-box-back -n memory-box-prod`.

## Estructura

```
memory-box-n8n/
├── docker-compose.yml
├── k8s/microk8s/          # base + overlays/prod para mark1
├── README.md
└── workflows/
    ├── new-order-notify.json      # Nuevo pedido → WhatsApp admin
    └── order-finalized-notify.json # Pedido finalizado → WhatsApp cliente
```
