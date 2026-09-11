# Automação Field Control + WhatsApp — n8n

Integração entre o Field Control (gestão de ordens de serviço) e o WhatsApp, usando n8n como motor de automação. Desenvolvido para empresas de assistência técnica.

> **Projeto desenvolvido entre 04/06/2026 e 17/06/2026.**

## O que faz

| Gatilho | Ação automática |
|---|---|
| Técnico conclui atividade | Cliente recebe link do relatório via WhatsApp |
| Orçamento criado | Cliente recebe link do orçamento + instruções de assinatura |
| Técnico inicia deslocamento | Cliente recebe link de rastreamento em tempo real |

---

## Arquitetura

```
Field Control ──► ngrok ──► n8n ──► WTS Chat ──► WhatsApp do cliente
WAHA (WhatsApp) ─────────────────────────────────────────────────────────────►
```

---

## Stack

- **[n8n](https://n8n.io)** — Motor de automação (self-hosted, gratuito)
- **[Docker Desktop](https://docker.com/products/docker-desktop)** — Executa o n8n e o WAHA
- **[ngrok](https://ngrok.com)** — URL pública fixa para receber webhooks
- **[WAHA](https://waha.devlike.pro)** — API HTTP para WhatsApp (self-hosted, gratuito)
- **Field Control API** — `https://carchost.fieldcontrol.com.br`
- **Field Control GraphQL** — `https://grond-2.fieldcontrol.com.br/graphql`
- **WTS Chat** — Nó n8n para envio de mensagens via WhatsApp (`n8n-nodes-wts`). Usado aqui através do Synova, um CRM brasileiro para o WhatsApp — mas pode ser substituído por qualquer plataforma compatível.

---

## Instalação

### 1. Pré-requisitos
- Docker Desktop instalado
- Conta gratuita no [ngrok](https://ngrok.com) com domínio fixo gerado

### 2. Subir o n8n

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  --restart unless-stopped \
  n8nio/n8n
```

Acesse em: `http://localhost:5678`

### 3. Subir o WAHA

Crie um arquivo `.env`:

```env
WAHA_DASHBOARD_USERNAME=admin
WAHA_DASHBOARD_PASSWORD=sua_senha
WHATSAPP_SWAGGER_USERNAME=admin
WHATSAPP_SWAGGER_PASSWORD=sua_senha
WAHA_API_KEY=sua_api_key
```

Depois rode:

```bash
docker run -d \
  --name waha \
  -p 3000:3000 \
  --env-file .env \
  --restart unless-stopped \
  devlikeapro/waha
```

Acesse o dashboard em: `http://localhost:3000/dashboard`

### 4. Configurar o ngrok

```bash
ngrok config add-authtoken SEU_TOKEN
```

No arquivo `ngrok.yml`, adicione:

```yaml
tunnels:
  n8n:
    proto: http
    addr: 5678
    domain: seu-dominio.ngrok-free.dev
```

Instale como serviço (Windows):

```bash
ngrok service install --config="C:\Users\SEU_USUARIO\AppData\Local\ngrok\ngrok.yml"
ngrok service start
```

### 5. Instalar o nó WTS Chat no n8n

No n8n: **Settings → Community Nodes → Install**

```
n8n-nodes-wts
```

### 6. Importar o workflow

Baixe o arquivo `workflow.json` deste repositório e importe no n8n via **Import from file**.

---

## Configuração

### Field Control
1. Gere uma API Key em **Configurações → API de Integração → Nova Chave**
2. Configure o webhook com a URL: `https://seu-dominio.ngrok-free.dev/webhook/SEU_WEBHOOK_ID`
3. Eventos: **Atividade concluída com sucesso** e **Criação de um orçamento**

### WAHA
1. Acesse o dashboard e crie uma sessão chamada exatamente `default`
2. Escaneie o QR Code com o número dedicado
3. Configure o webhook da sessão: URL = `https://seu-dominio.ngrok-free.dev/webhook/SEU_WEBHOOK_WAHA_ID`, Evento: `message`

### n8n
Configure as credenciais:
- **Header Auth** com `X-Api-Key` para o Field Control
- **WTS account** com o token Bearer do Synova

---

## Descobertas técnicas relevantes

### IDs do Field Control são Base64
O `order.id` vem codificado em Base64 no formato `uuid:accountId`. Para montar links:
```javascript
const decoded = Buffer.from(id, 'base64').toString('utf-8');
const realId = decoded.split(':')[0];
```

### Headers chegam em minúsculas
O header do evento chega como `x-fieldcontrol-event` (não `X-FieldControl-Event`).

### Link de rastreamento via GraphQL
O link de deslocamento do técnico não é exposto via API REST. Porém, a API GraphQL pública resolve o `trackingId` para um JWT com os dados da atividade:

```javascript
// Query GraphQL (sem autenticação necessária)
{
  "operationName": "TaskTrackingTokenQuery",
  "variables": { "id": "TRACKING_ID" },
  "query": "query TaskTrackingTokenQuery($id: ID!) { taskTrackingToken(id: $id) }"
}

// Decodificar o JWT retornado
const payload = token.split('.')[1];
const data = JSON.parse(Buffer.from(payload, 'base64').toString('utf-8'));
// data.taskId → ID da atividade
// data.accountId → ID da conta
```

---

## Estrutura do workflow

```
Webhook (Field Control)
    └── Switch (por x-fieldcontrol-event)
            ├── task-completed     → busca OS → busca cliente → envia relatório
            ├── task-route-started → (não usado nesta trilha)
            └── quotation-created  → monta link → busca cliente → envia orçamento + imagem

Webhook (WAHA — grupo dos técnicos)
    └── Filter (filtra pelo ID do grupo)
            └── Code (extrai trackingId)
                    └── GraphQL → decodifica JWT → busca atividade → OS → cliente → envia link
```

---

## Limitações conhecidas

- **WAHA Core (gratuito):** suporta apenas 1 sessão chamada `default`
- **n8n self-hosted:** não exibe output de nós em execuções de produção no canvas
- **Uptime:** depende do PC estar ligado. Para 24/7, considere migrar para VPS
- **ngrok gratuito:** limite de 20.000 requisições/mês

---

## Licença

MIT — sinta-se livre para usar, adaptar e contribuir.
