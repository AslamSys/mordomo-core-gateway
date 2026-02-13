# 🚪 Core Gateway

**Container:** `core-gateway`  
**Ecossistema:** Mordomo  
**Posição no Fluxo:** Gateway Central

---

## 📋 Propósito

Gateway REST + WebSocket que expõe APIs para o Aslam App (interface visual no tablet) e coordena requisições externas. NÃO é a entrada primária de voz (isso é feito via microfone USB).

---

## 🎯 Responsabilidades

### Primárias
- ✅ Expor REST API para Aslam App (tablet)
- ✅ WebSocket para updates em tempo real (enviar info visual ao tablet)
- ✅ Enviar comandos ao tablet quando Mordomo quer MOSTRAR algo
- ✅ Rate limiting por speaker_id
- ✅ CORS e segurança de endpoints
- ✅ Roteamento de requisições para serviços internos
- ✅ Health checks de todos os containers

### Secundárias
- ✅ Métricas de requisições (Prometheus)
- ✅ Logging estruturado de acessos
- ✅ Circuit breaker para serviços downstream
- ✅ Request/Response caching (Redis)

---

## 🔧 Tecnologias

**Stack:** Node.js + TypeScript

```json
{
  "express": "^4.18.2",
  "ws": "^8.14.2",
  "helmet": "^7.1.0",
  "cors": "^2.8.5",
  "express-rate-limit": "^7.1.5",
  "winston": "^3.11.0",
  "prom-client": "^15.1.0"
}
```

---

## 📊 Especificações

```yaml
Performance:
  CPU: 5-10%
  RAM: ~ 150 MB
  Concurrent Connections: 10
  Request Latency: < 50 ms
  WebSocket Latency: < 10 ms
  
Scalability:
  Horizontal: Sim (stateless com Redis)
  Load Balancer: Ready
```

---

## 🔌 Interfaces

### REST API (Aslam App - Tablet)

```typescript
// Conversas
GET    /api/v1/conversations
  Query: ?speaker_id=user_1&limit=50
  Response: { conversations: [...], total: 123 }

GET    /api/v1/conversations/:id
  Response: { id, speaker_id, messages: [...], started_at, ended_at }

POST   /api/v1/conversations/:id/messages
  Body: { text: "Qual a temperatura?" }
  Response: { message_id, status: "processing" }

DELETE /api/v1/conversations/:id
  Response: { deleted: true }

// Status do Sistema
GET    /api/v1/status
  Response: {
    mordomo: { containers: [...], health: "healthy" },
    infraestrutura: { nats: "up", postgres: "up", ... },
    monitoramento: { ... }
  }

GET    /api/v1/health
  Response: { status: "ok", uptime: 12345 }

// Métricas
GET    /api/v1/metrics
  Response: Prometheus format

// Usuários
GET    /api/v1/users
  Response: [{ id, name, speaker_id, last_seen }]

GET    /api/v1/users/:id/history
  Query: ?limit=100&offset=0
  Response: { messages: [...], total: 456 }
```

### WebSocket (Real-time Updates para Tablet)

```typescript
// Conexão do Aslam App (tablet)
ws://core-gateway:3000/ws

// Eventos enviados ao tablet (quando Mordomo quer mostrar algo)
{
  "event": "conversation.started",
  "data": { conversation_id: "uuid", speaker_id: "user_1" }
}

{
  "event": "message.received",
  "data": { 
    conversation_id: "uuid",
    role: "user",
    content: "Acende a luz",
    timestamp: 1732723200
  }
}

{
  "event": "message.response",
  "data": {
    conversation_id: "uuid",
    role: "assistant",
    content: "Luz acesa!",
    confidence: 0.95
  }
}

{
  "event": "tts.playing",
  "data": { conversation_id: "uuid", progress: 0.5 }
}

{
  "event": "container.status",
  "data": { "container": "whisper-asr", "status": "healthy" }
}

// Mordomo solicita exibição visual
{
  "event": "ui.show_chart",
  "data": {
    "type": "energy_consumption",
    "period": "month",
    "chart_data": [...]
  }
}

{
  "event": "ui.show_camera",
  "data": {
    "camera_id": "cam_entrada",
    "stream_url": "rtsp://..."
  }
}
```

### NATS Subscriptions (Recebe de outros containers)

```yaml
# Transcrições processadas
mordomo.conversation.message_received:
  payload:
    conversation_id: uuid
    speaker_id: user_1
    text: "Qual a temperatura?"
    timestamp: 1732723200.123

# Resposta do Brain
mordomo.conversation.response_generated:
  payload:
    conversation_id: uuid
    response: "A temperatura é 23°C"
    confidence: 0.95
    actions: []

# Status de containers
mordomo.container.status_changed:
  payload:
    container: whisper-asr
    status: healthy | degraded | down
    timestamp: 1732723200.123
```

### NATS Publications (Envia para outros containers)

```yaml
# Requisição de conversação
mordomo.conversation.start_requested:
  payload:
    conversation_id: uuid
    speaker_id: user_1
    timestamp: 1732723200.123

# Envio de mensagem
mordomo.conversation.message_sent:
  payload:
    conversation_id: uuid
    speaker_id: user_1
    text: "Acende a luz"
```

---

## ⚙️ Configuração

```yaml
server:
  port: 3000
  host: "0.0.0.0"
  
cors:
  enabled: true
  origins:
    - "http://localhost:*"
    - "http://192.168.1.*"  # Rede local
  
rate_limit:
  window_ms: 60000  # 1 minuto
  max_requests: 100
  per_speaker: true
  store: "memory"  # In-memory (sem Redis)
  
nats:
  url: "nats://nats:4222"
  max_reconnect: 10
  
downstream_services:
  conversation_manager:
    url: "http://conversation-manager:3001"
    timeout: 5000
  action_dispatcher:
    url: "http://action-dispatcher:3002"
    timeout: 3000

security:
  helmet:
    enabled: true
    hsts: true
    noSniff: true
    xssFilter: true
  
  # Sem login - apenas Speaker Verification por voz
  speaker_verification:
    enabled: true
    # Voices autorizadas via speaker-verification container
```

---

## 🔒 Segurança

### Secrets Management

**Opção 1: Docker Secrets (Recomendado para setup simples)**

```yaml
# docker-compose.yml
services:
  core-gateway:
    secrets:
      - redis_password
      - nats_token

secrets:
  redis_password:
    file: ./secrets/redis_password.txt
  nats_token:
    file: ./secrets/nats_token.txt
```

```typescript
// Acessar no código
import fs from 'fs';

const redisPassword = fs.readFileSync('/run/secrets/redis_password', 'utf8').trim();
```

**Opção 2: HashiCorp Vault (Produção)**

```yaml
vault:
  enabled: false  # Desabilitado para uso doméstico
  url: "http://vault:8200"
  token: "${VAULT_TOKEN}"
  path: "secret/mordomo/core-gateway"
```

**Variáveis de Ambiente (.env) - NÃO COMMITAR**

```bash
# .env (gitignored)
REDIS_PASSWORD=sua_senha_redis_aqui
NATS_TOKEN=seu_token_nats_aqui
POSTGRES_PASSWORD=sua_senha_postgres_aqui
```

### Proteções Implementadas

- ✅ **Helmet.js:** Headers de segurança HTTP
- ✅ **CORS:** Apenas rede local (192.168.x.x)
- ✅ **Rate Limiting:** 100 req/min por speaker
- ✅ **Input Validation:** Sanitização de inputs
- ✅ **No Authentication:** Speaker Verification via voz suficiente
- ✅ **Secrets:** Docker Secrets ou variáveis de ambiente

---

## 📡 Comunicação com Outros Módulos

### Gateway → Outros Módulos (via NATS Request/Reply)

```typescript
// Enviar comando para IoT
const response = await nats.request(
  'iot.command.execute',
  {
    device_id: 'light_sala',
    action: 'turn_on',
    speaker_id: 'user_1'
  },
  { timeout: 3000 }
);

// Enviar mensagem WhatsApp via Comunicação
await nats.request(
  'comunicacao.whatsapp.send',
  {
    to: '+5511999999999',
    message: 'Lembrete: Reunião às 15h',
    speaker_id: 'user_1'
  }
);

// Fazer PIX via Pagamentos
await nats.request(
  'pagamentos.pix.send',
  {
    to_key: 'email@example.com',
    amount: 50.00,
    description: 'Pagamento teste',
    speaker_id: 'user_1'
  }
);
```

### Outros Módulos → Gateway (Notificações)

```typescript
// IoT notifica mudança de estado
nats.subscribe('iot.device.state_changed', (msg) => {
  // Envia via WebSocket para Dashboard
  wsBroadcast({
    event: 'iot.device.updated',
    data: msg.data
  });
});

// Pagamentos notifica conclusão de PIX
nats.subscribe('pagamentos.pix.completed', (msg) => {
  wsBroadcast({
    event: 'payment.completed',
    data: msg.data
  });
});
```

---

## 🐳 Docker

```yaml
services:
  core-gateway:
    build: ./core-gateway
    container_name: mordomo-core-gateway
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - NATS_URL=nats://nats:4222
    secrets:
      - nats_token
    depends_on:
      - nats
    networks:
      - mordomo-net
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 200M
        reservations:
          cpus: '0.1'
          memory: 100M
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/api/v1/health"]
      interval: 30s
      timeout: 5s
      retries: 3

secrets:
  redis_password:
    file: ./secrets/redis_password.txt
  nats_token:
    file: ./secrets/nats_token.txt
```

---

## 📝 Exemplo de Código

```typescript
// src/server.ts
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import rateLimit from 'express-rate-limit';
import { WebSocketServer } from 'ws';
import Redis from 'ioredis';
import { connect, NatsConnection } from 'nats';
import { readFileSync } from 'fs';

const app = express();
const port = process.env.PORT || 3000;

// Middleware
app.use(helmet());
app.use(cors({ origin: /^http:\/\/(localhost|192\.168\.\d+\.\d+)/ }));
app.use(express.json());

// Rate limiting
const limiter = rateLimit({
  windowMs: 60000,
  max: 100,
  keyGenerator: (req) => req.headers['x-speaker-id'] || req.ip
});
app.use('/api', limiter);

// Redis (cache)
const redis = new Redis({
  host: 'redis',
  port: 6379,
  password: readFileSync('/run/secrets/redis_password', 'utf8').trim()
});

// NATS (event bus)
let nats: NatsConnection;
(async () => {
  nats = await connect({
    servers: 'nats://nats:4222',
    token: readFileSync('/run/secrets/nats_token', 'utf8').trim()
  });
  console.log('✅ NATS connected');
})();

// REST API
app.get('/api/v1/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

app.get('/api/v1/conversations', async (req, res) => {
  const { speaker_id, limit = 50 } = req.query;
  
  // Cache check
  const cacheKey = `conversations:${speaker_id}:${limit}`;
  const cached = await redis.get(cacheKey);
  if (cached) {
    return res.json(JSON.parse(cached));
  }
  
  // Request to conversation-manager
  const response = await nats.request(
    'mordomo.conversation.list',
    { speaker_id, limit },
    { timeout: 3000 }
  );
  
  const data = JSON.parse(new TextDecoder().decode(response.data));
  
  // Cache for 5 min
  await redis.setex(cacheKey, 300, JSON.stringify(data));
  
  res.json(data);
});

// WebSocket
const wss = new WebSocketServer({ server: app.listen(port) });

wss.on('connection', (ws, req) => {
  const speakerId = new URL(req.url!, `http://${req.headers.host}`).searchParams.get('speaker_id');
  
  console.log(`WebSocket connected: ${speakerId}`);
  
  // Subscribe to user-specific events
  const sub = nats.subscribe(`mordomo.conversation.>${speakerId}`);
  
  (async () => {
    for await (const msg of sub) {
      ws.send(JSON.stringify({
        event: msg.subject,
        data: JSON.parse(new TextDecoder().decode(msg.data))
      }));
    }
  })();
  
  ws.on('close', () => {
    sub.unsubscribe();
    console.log(`WebSocket disconnected: ${speakerId}`);
  });
});

console.log(`🚀 Core Gateway running on port ${port}`);
```

---

## 📊 Métricas (Prometheus)

```typescript
import client from 'prom-client';

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency',
  labelNames: ['method', 'route', 'status']
});

const wsConnections = new client.Gauge({
  name: 'websocket_connections_active',
  help: 'Active WebSocket connections'
});

// Endpoint
app.get('/api/v1/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

---

## 🔧 Troubleshooting

### WebSocket não conecta
```bash
# Verificar porta aberta
docker exec core-gateway netstat -tuln | grep 3000

# Testar conexão
wscat -c ws://localhost:3000/ws?speaker_id=user_1
```

### Alta latência
```bash
# Verificar cache Redis
docker exec core-gateway redis-cli --raw -a $(cat secrets/redis_password.txt) INFO stats

# Ver hit rate
GET /api/v1/metrics | grep cache_hit_rate
```

### Rate limit atingido
```bash
# Ajustar limites
vim config/rate-limit.yml
# Reiniciar
docker restart core-gateway
```

---

**Documentação atualizada:** 27/11/2025
