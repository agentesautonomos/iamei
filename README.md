# Agente WhatsApp com IA

Agente de IA para WhatsApp construído com Claude (Anthropic), FastAPI, Redis e Z-API.

## Funcionalidades base

- Responde mensagens de texto
- Transcreve áudios automaticamente (Groq Whisper)
- Lê e analisa PDFs e planilhas Excel
- Memória persistente por usuário (Redis)
- Painel admin com visualização de conversas
- Prompt editável pelo painel — sem redeploy
- Arquivos de referência injetados no prompt automaticamente

## Funcionalidades de monetização (opcionais)

- **Freemium/Premium** — controle de acesso por número de WhatsApp
- **Assinaturas mensais** — integração com Mercado Pago (planos recorrentes)
- **Agendamento de consultas/sessões** — calendário com slots disponíveis, bloqueio de datas
- **Pagamento de sessão avulsa** — Checkout Pro do Mercado Pago
- **Confirmação automática por WhatsApp** — após pagamento aprovado
- **Painel de agenda** — CRUD completo, bloqueio de datas e horários, configuração de valores

---

## Variáveis de ambiente

### Obrigatórias

| Variável | Descrição |
|----------|-----------|
| ANTHROPIC_API_KEY | Chave da API da Anthropic |
| GROQ_API_KEY | Chave da API do Groq (transcrição de áudio) |
| AGENT_NAME | Nome do agente |
| AGENT_MODEL | Modelo do Claude (padrão: claude-haiku-4-5-20251001) |
| ZAPI_INSTANCE_ID | Instance ID da Z-API |
| ZAPI_TOKEN | Token da Z-API |
| ZAPI_CLIENT_TOKEN | Client Token da Z-API |
| REDIS_URL | URL do Redis |
| ADMIN_USER | Login do painel admin |
| ADMIN_PASS | Senha do painel admin |
| BASE_URL | URL pública do servidor (ex: https://seuagente.up.railway.app) |

### Opcionais — Monetização com Mercado Pago

| Variável | Descrição |
|----------|-----------|
| MP_ACCESS_TOKEN | Access Token da aplicação de Assinaturas (MP) |
| MP_PUBLIC_KEY | Public Key da aplicação de Assinaturas (MP) |
| MP_PLAN_ID | ID do plano de assinatura criado no MP |
| MP_ACCESS_TOKEN_CONSULTA | Access Token da aplicação de Checkout Pro (MP) |
| LINK_PAGAMENTO | Link externo de pagamento (opcional, sobrescreve o gerado automaticamente) |

---

## Deploy

Start Command:
```
uvicorn main:app --host 0.0.0.0 --port $PORT
```

---

## Painel admin

Acesse `/admin` com o login e senha configurados nas variáveis.

| Rota | Descrição |
|------|-----------|
| `/admin` | Lista de usuários com status freemium/premium |
| `/admin/assinaturas` | Gerenciar assinaturas — ativar, desativar, criar manualmente |
| `/admin/agenda` | Gerenciar agendamentos, bloqueios e configurações |
| `/admin/consultas` | Interessados em consulta que o bot registrou |
| `/admin/prompt` | Editar o prompt do agente |
| `/admin/arquivos` | Upload de arquivos de referência para o prompt |

---

## Rotas públicas

| Rota | Descrição |
|------|-----------|
| `/` | Status do agente |
| `/pagamento` | Página de assinatura Premium |
| `/pagamento/obrigado` | Retorno após pagamento da assinatura |
| `/consulta` | Página de agendamento de sessão/consulta |
| `/consulta/checkout` | Cria preferência no MP e redireciona para pagamento |
| `/consulta/obrigado` | Retorno após pagamento da sessão |
| `/webhook` | Webhook da Z-API (mensagens WhatsApp) |
| `/webhook/mercadopago` | Webhook do Mercado Pago (pagamentos e assinaturas) |

---

## Como ativar a monetização

### 1. Criar as aplicações no Mercado Pago

Acesse developers.mercadopago.com.br e crie duas aplicações separadas:

**Aplicação 1 — Assinaturas:**
- Produto: Assinaturas (Suscripciones)
- Copie o Access Token e o Public Key
- Crie o plano de assinatura e copie o Plan ID

**Aplicação 2 — Consultas/Sessões avulsas:**
- Produto: Checkout Pro
- Copie o Access Token

### 2. Adicionar variáveis no Railway

```
MP_ACCESS_TOKEN=seu_token_assinaturas
MP_PUBLIC_KEY=sua_public_key
MP_PLAN_ID=id_do_plano
MP_ACCESS_TOKEN_CONSULTA=seu_token_checkout_pro
```

### 3. Configurar webhooks no Mercado Pago

Em cada aplicação, vá em Webhooks e adicione:
```
https://SEU-DOMINIO.up.railway.app/webhook/mercadopago
```

Para a aplicação de Assinaturas: ative eventos de **Assinaturas**.
Para a aplicação de Consultas: ative eventos de **Pagamentos**.

### 4. Configurar o prompt com {STATUS} e [LINK_PAGAMENTO]

O sistema injeta automaticamente no prompt:
- `{STATUS}` → `FREEMIUM` ou `PREMIUM` conforme o número do usuário
- `[LINK_PAGAMENTO]` → link de assinatura gerado automaticamente

Exemplo de trecho de prompt para freemium:
```
Quando STATUS = FREEMIUM, entregue uma dica gratuita e convide para o Premium:
"Para orientacoes completas, acesse o plano Premium: [LINK_PAGAMENTO]"
```

### 5. Configurar a agenda

Acesse `/admin/agenda` e:
- Defina os horários disponíveis
- Configure o valor da sessão
- Bloqueie datas ou horários indisponíveis

No prompt, adicione o link da agenda para o bot enviar quando solicitado:
```
Se o usuario quiser agendar uma sessao, envie:
"Para agendar, acesse: https://SEU-DOMINIO.up.railway.app/consulta"
```

---

## Fluxo completo de pagamento

### Assinatura mensal
1. Bot envia `[LINK_PAGAMENTO]` para o usuário
2. Usuário assina no Mercado Pago
3. MP envia webhook para `/webhook/mercadopago`
4. Sistema ativa o número automaticamente
5. Bot envia mensagem de boas-vindas Premium

### Sessão avulsa
1. Bot envia link `/consulta` para o usuário
2. Usuário escolhe data e horário na página
3. Sistema reserva o slot e redireciona para o MP
4. Usuário paga via Checkout Pro
5. Sistema confirma o agendamento e envia detalhes por WhatsApp

---

## Estrutura de dados no Redis

| Chave | Descrição |
|-------|-----------|
| `historico:{telefone}` | Histórico de mensagens por usuário |
| `assinatura:{telefone}` | Status e dados da assinatura |
| `agendamento:{data}_{hora}` | Dados de cada agendamento |
| `bloqueio:{data}_{hora}` | Horário bloqueado |
| `bloqueio:{data}_dia` | Dia inteiro bloqueado |
| `consulta:{telefone}` | Interesse em consulta registrado pelo bot |
| `config:agent_prompt` | Prompt editável pelo painel |
| `config:agenda` | Configurações de horários e valor |
| `config:arquivo:{nome}` | Arquivos de referência do prompt |
