# API Externa de Agendamentos - Clínica Oitava Rosado

API REST para integração de sistemas externos (bots de WhatsApp, IA, parceiros) com o módulo de agendamentos da clínica.

## Visão Geral

| Item | Detalhe |
|------|---------|
| **Base URL** | `https://{seu-dominio}/api/v1/external` |
| **Autenticação** | API Key via header `X-API-Key` |
| **Formato** | JSON |
| **Rate Limit** | 60 requisições/minuto (configurável por chave) |
| **Webhooks** | Notificações automáticas via POST (HMAC-SHA256) |

## Índice

- [Autenticação](#autenticação)
- [Formato de Resposta](#formato-de-resposta)
- [Endpoints](#endpoints)
  - [Agendas](#agendas)
  - [Pacientes](#pacientes)
  - [Agendamentos](#agendamentos)
  - [Convênios](#convênios)
- [Webhooks](#webhooks)
- [Permissões](#permissões)
- [Códigos de Erro](#códigos-de-erro)
- [Exemplos de Integração](#exemplos-de-integração)

---

## Autenticação

Todas as requisições devem incluir o header `X-API-Key` com uma chave válida.

```bash
curl -H "X-API-Key: clk_sua_chave_aqui" \
  https://seu-dominio/api/v1/external/agendas
```

### Obter uma API Key

API Keys são geradas pelo administrador do sistema via linha de comando:

```bash
php artisan apikey:manage create --name="Bot WhatsApp" --permissions="*"
```

### Respostas de Autenticação

| Código | Situação |
|--------|----------|
| `401` | Chave não fornecida ou inválida |
| `403` | Chave válida, mas sem permissão para o recurso |
| `429` | Rate limit excedido |

---

## Formato de Resposta

Todas as respostas seguem o formato:

### Sucesso
```json
{
  "sucesso": true,
  "dados": { ... },
  "mensagem": "Descrição do resultado"
}
```

### Erro
```json
{
  "sucesso": false,
  "mensagem": "Descrição do erro",
  "erros": { ... }
}
```

---

## Endpoints

### Agendas

#### Listar Agendas Ativas

```
GET /agendas
```

Retorna todas as agendas ativas (consultas e procedimentos).

**Permissão:** `agendas.read`

**Parâmetros de Query (opcionais):**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `tipo` | string | Filtrar por tipo: `consulta` ou `procedimento` |
| `specialty_id` | integer | Filtrar por especialidade |
| `provider_id` | integer | Filtrar por profissional |
| `exam_group_id` | integer | Filtrar por grupo de exames |

**Exemplo:**
```bash
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/agendas?tipo=consulta"
```

**Resposta:**
```json
{
  "sucesso": true,
  "dados": [
    {
      "id": 1,
      "tipo": "consulta",
      "tipo_agenda": "horario_marcado",
      "sala": "Consultório 01",
      "telefone": "(84) 3322-1100",
      "tempo_estimado_minutos": 30,
      "limite_vagas_dia": 20,
      "aceita_cartao_desconto": false,
      "orientacoes": "Chegar 15 minutos antes",
      "provider": { "id": 1, "name": "Dr. João Silva" },
      "specialty": { "id": 5, "name": "Cardiologia" },
      "exam_group": null,
      "unit": { "id": 1, "name": "Mossoró" }
    }
  ],
  "mensagem": "Agendas listadas com sucesso"
}
```

---

#### Detalhes da Agenda

```
GET /agendas/{id}
```

Retorna detalhes completos incluindo horários de funcionamento e convênios aceitos.

**Permissão:** `agendas.read`

**Resposta:**
```json
{
  "sucesso": true,
  "dados": {
    "id": 1,
    "tipo": "consulta",
    "tipo_agenda": "horario_marcado",
    "sala": "Consultório 01",
    "telefone": "(84) 3322-1100",
    "tempo_estimado_minutos": 30,
    "limite_vagas_dia": 20,
    "limite_encaixes_dia": 3,
    "limite_retornos_dia": 5,
    "idade_minima": null,
    "aceita_cartao_desconto": false,
    "orientacoes": "Chegar 15 minutos antes",
    "informacoes_fixas": null,
    "provider": { "id": 1, "name": "Dr. João Silva" },
    "specialty": { "id": 5, "name": "Cardiologia" },
    "exam_group": null,
    "unit": { "id": 1, "name": "Mossoró" },
    "horarios": [
      {
        "dia_semana": "Segunda",
        "horario_inicio_manha": "07:00",
        "horario_fim_manha": "12:00",
        "horario_inicio_tarde": "13:00",
        "horario_fim_tarde": "17:00",
        "vagas_dia": 20,
        "aceita_sedacao": false,
        "limite_sedacoes": null
      }
    ],
    "convenios": [
      {
        "convenio_id": 1,
        "nome_convenio": "Unimed",
        "limite_atendimentos": null
      }
    ]
  },
  "mensagem": "Agenda encontrada"
}
```

---

#### Consultar Disponibilidade

```
GET /agendas/{id}/disponibilidade?data=YYYY-MM-DD
```

Retorna os horários disponíveis para uma data específica.

**Permissão:** `agendas.read`

**Parâmetros obrigatórios:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `data` | date | Data no formato `YYYY-MM-DD` |

**Exemplo:**
```bash
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/agendas/1/disponibilidade?data=2026-03-10"
```

**Resposta (agenda normal com slots fixos):**
```json
{
  "sucesso": true,
  "dados": {
    "disponivel": true,
    "dia_semana": "Segunda",
    "slots": [
      { "horario": "07:00", "status": "disponivel" },
      { "horario": "07:30", "status": "ocupado" },
      { "horario": "08:00", "status": "disponivel" },
      { "horario": "08:30", "status": "bloqueado" }
    ],
    "vagas_total": 20,
    "vagas_disponiveis": 15,
    "encaixes_disponiveis": 3,
    "retornos_disponiveis": 5,
    "aceita_sedacao": false,
    "sedacoes_disponiveis": 0
  },
  "mensagem": "Disponibilidade consultada"
}
```

**Resposta (Ressonância Magnética — modo sequencial):**
```json
{
  "sucesso": true,
  "dados": {
    "disponivel": true,
    "modo_rm": true,
    "dia_semana": "Segunda",
    "vagas_total": 10,
    "vagas_ocupadas": 3,
    "encaixes_disponiveis": 2,
    "retornos_disponiveis": 3,
    "aceita_sedacao": true,
    "sedacoes_disponiveis": 2,
    "proximo_horario": "09:15",
    "fim_expediente": "18:00",
    "minutos_restantes": 525
  },
  "mensagem": "Disponibilidade consultada"
}
```

> **Nota:** Agendas de Ressonância Magnética (exam_group_id=34) utilizam horários sequenciais calculados automaticamente. O campo `proximo_horario` indica o próximo horário disponível.

**Situações especiais:**
- **Agenda não atende no dia:** `disponivel: false`, `slots: []`
- **Agenda bloqueada:** `disponivel: false`, `bloqueada: true`, `mensagem` com motivo

---

### Pacientes

#### Buscar Paciente

```
GET /pacientes/buscar
```

Busca pacientes por CPF, telefone ou nome. Informe **pelo menos um** filtro.

**Permissão:** `pacientes.read`

**Parâmetros de Query:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `cpf` | string | CPF (com ou sem formatação) — busca exata |
| `telefone` | string | Telefone — busca parcial |
| `nome` | string | Nome — busca parcial (case insensitive) |

**Prioridade:** CPF > Telefone > Nome (usa o primeiro encontrado)

**Exemplo:**
```bash
# Por CPF
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/pacientes/buscar?cpf=12345678910"

# Por telefone
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/pacientes/buscar?telefone=84987654321"

# Por nome
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/pacientes/buscar?nome=Maria"
```

**Resposta:**
```json
{
  "sucesso": true,
  "dados": [
    {
      "id": 1,
      "name": "Maria Silva Santos",
      "nome_social": null,
      "cpf": "12345678910",
      "phone": "(84) 98765-4321",
      "email": "maria@email.com",
      "birth_date": "1985-03-15",
      "gender": "F"
    }
  ],
  "mensagem": "Busca realizada"
}
```

---

#### Cadastrar Paciente

```
POST /pacientes
```

Cadastra um novo paciente com dados básicos.

**Permissão:** `pacientes.write`

**Body (JSON):**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome completo |
| `phone` | string | Sim | Telefone com DDD |
| `birth_date` | date | Sim | Data de nascimento (`YYYY-MM-DD`) |
| `gender` | string | Sim | Sexo: `M`, `F` ou `O` |
| `cpf` | string | Não | CPF (verificação de duplicidade) |
| `email` | string | Não | E-mail |
| `nome_social` | string | Não | Nome social |

**Exemplo:**
```bash
curl -X POST -H "X-API-Key: clk_..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "João da Silva",
    "phone": "(84) 99999-1234",
    "birth_date": "1990-05-20",
    "gender": "M",
    "cpf": "98765432100"
  }' \
  "https://seu-dominio/api/v1/external/pacientes"
```

**Resposta (201):**
```json
{
  "sucesso": true,
  "dados": {
    "id": 42,
    "name": "João da Silva",
    "nome_social": null,
    "cpf": "98765432100",
    "phone": "(84) 99999-1234",
    "email": null,
    "birth_date": "1990-05-20",
    "gender": "M"
  },
  "mensagem": "Paciente cadastrado com sucesso"
}
```

**Erro de CPF duplicado (422):**
```json
{
  "sucesso": false,
  "mensagem": "Já existe um paciente com este CPF",
  "erros": { "paciente_existente_id": 15 }
}
```

---

#### Dados do Paciente

```
GET /pacientes/{id}
```

**Permissão:** `pacientes.read`

**Resposta:**
```json
{
  "sucesso": true,
  "dados": {
    "id": 1,
    "name": "Maria Silva Santos",
    "nome_social": null,
    "cpf": "12345678910",
    "phone": "(84) 98765-4321",
    "email": "maria@email.com",
    "birth_date": "1985-03-15",
    "gender": "F",
    "blood_type": "O+",
    "allergies": "Dipirona",
    "address": "Rua das Flores, 123",
    "city": "Natal",
    "state": "RN"
  },
  "mensagem": "Paciente encontrado"
}
```

---

#### Agendamentos do Paciente

```
GET /pacientes/{id}/agendamentos
```

Retorna os **próximos** agendamentos (futuros, não cancelados).

**Permissão:** `agendamentos.read`

**Resposta:**
```json
{
  "sucesso": true,
  "dados": [
    {
      "id": 15,
      "numero": "AGD-0015",
      "data": "2026-03-10",
      "hora": "08:00",
      "status": "AGENDADO",
      "tipo_agendamento": "NORMAL",
      "confirmado": false,
      "observacoes": null,
      "agenda": {
        "id": 1,
        "provider": "Dr. João Silva",
        "specialty": "Cardiologia",
        "exam_group": null,
        "unit": "Mossoró"
      },
      "convenio": "Unimed",
      "procedimentos": ["Consulta Cardiológica"]
    }
  ],
  "mensagem": "Agendamentos do paciente"
}
```

---

### Agendamentos

#### Criar Agendamento

```
POST /agendamentos
```

Cria um novo agendamento. Dispara webhook `agendamento.criado`.

**Permissão:** `agendamentos.write`

**Body (JSON):**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `agenda_id` | integer | Sim | ID da agenda |
| `data_agendamento` | date | Sim | Data (`YYYY-MM-DD`, hoje ou futuro) |
| `paciente_id` | integer | Condicional | ID do paciente existente |
| `nome_paciente` | string | Condicional | Nome (se não informar paciente_id) |
| `telefone_paciente` | string | Condicional | Telefone (se não informar paciente_id) |
| `hora_agendamento` | string | Condicional | Horário `HH:MM` (obrigatório exceto para RM) |
| `convenio_id` | integer | Não | ID do convênio |
| `tipo_agendamento` | string | Não | `NORMAL` (padrão), `ENCAIXE` ou `RETORNO` |
| `tipo_consulta` | string | Não | `primeira_vez` (padrão) ou `retorno` |
| `precisa_sedacao` | boolean | Não | Se necessita sedação (RM) |
| `observacoes` | string | Não | Observações (máx 1000 chars) |
| `procedure_ids` | array | Não | IDs dos procedimentos |

> **Nota:** Informe `paciente_id` **OU** `nome_paciente` + `telefone_paciente`.
>
> **Ressonância Magnética:** Se a agenda for de RM (exam_group_id=34) e `hora_agendamento` não for informado, o horário é calculado automaticamente baseado na fila sequencial.

**Exemplo — Agendamento com paciente existente:**
```bash
curl -X POST -H "X-API-Key: clk_..." \
  -H "Content-Type: application/json" \
  -d '{
    "agenda_id": 1,
    "paciente_id": 42,
    "data_agendamento": "2026-03-10",
    "hora_agendamento": "08:00",
    "convenio_id": 1,
    "tipo_consulta": "primeira_vez"
  }' \
  "https://seu-dominio/api/v1/external/agendamentos"
```

**Exemplo — Reserva de horário (sem paciente cadastrado):**
```bash
curl -X POST -H "X-API-Key: clk_..." \
  -H "Content-Type: application/json" \
  -d '{
    "agenda_id": 12,
    "nome_paciente": "Carlos Souza",
    "telefone_paciente": "(84) 99888-7777",
    "data_agendamento": "2026-03-10",
    "observacoes": "Agendado via WhatsApp"
  }' \
  "https://seu-dominio/api/v1/external/agendamentos"
```

**Resposta (201):**
```json
{
  "sucesso": true,
  "dados": {
    "id": 18,
    "numero": "AGD-0018",
    "data": "2026-03-10",
    "hora": "08:00",
    "status": "AGENDADO",
    "tipo_agendamento": "NORMAL",
    "tipo_consulta": "primeira_vez",
    "confirmado": null,
    "precisa_sedacao": false,
    "observacoes": null,
    "motivo_cancelamento": null,
    "paciente": {
      "id": 42,
      "name": "João da Silva",
      "phone": "(84) 99999-1234",
      "cpf": "98765432100"
    },
    "convenio": "Unimed",
    "procedimentos": [],
    "agenda": {
      "id": 1,
      "provider": "Dr. João Silva",
      "specialty": "Cardiologia",
      "exam_group": null,
      "unit": "Mossoró"
    }
  },
  "mensagem": "Agendamento criado com sucesso"
}
```

**Erros possíveis (422):**

| Mensagem | Causa |
|----------|-------|
| `Limite de vagas do dia atingido` | Todas as vagas NORMAL estão preenchidas |
| `Limite de encaixes do dia atingido` | Limite de encaixes atingido |
| `Limite de retornos do dia atingido` | Limite de retornos atingido |
| `Este horário já está ocupado` | Slot já possui agendamento |
| `Agenda bloqueada nesta data` | Agenda com bloqueio ativo |
| `Este dia não aceita sedação` | Sedação solicitada em dia sem suporte |
| `Limite de sedações do dia atingido` | Máximo de sedações atingido |
| `Exame com contraste requer médico presente` | Contraste sem médico no horário |
| `Não há mais horário disponível neste dia para RM` | RM sem espaço na fila |

---

#### Ver Agendamento

```
GET /agendamentos/{id}
```

**Permissão:** `agendamentos.read`

Retorna os mesmos dados do POST de criação.

---

#### Confirmar Agendamento

```
PUT /agendamentos/{id}/confirmar
```

Marca o agendamento como confirmado. Dispara webhook `agendamento.confirmado`.

**Permissão:** `agendamentos.write`

**Exemplo:**
```bash
curl -X PUT -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/agendamentos/18/confirmar"
```

**Resposta:**
```json
{
  "sucesso": true,
  "dados": {
    "id": 18,
    "status": "CONFIRMADO",
    "confirmado": true,
    "..."
  },
  "mensagem": "Agendamento confirmado com sucesso"
}
```

---

#### Cancelar Agendamento

```
PUT /agendamentos/{id}/cancelar
```

Cancela o agendamento. Dispara webhook `agendamento.cancelado`.

**Permissão:** `agendamentos.write`

**Body (JSON, opcional):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `motivo` | string | Motivo do cancelamento (máx 500 chars) |

**Exemplo:**
```bash
curl -X PUT -H "X-API-Key: clk_..." \
  -H "Content-Type: application/json" \
  -d '{"motivo": "Paciente solicitou cancelamento via WhatsApp"}' \
  "https://seu-dominio/api/v1/external/agendamentos/18/cancelar"
```

**Resposta:**
```json
{
  "sucesso": true,
  "dados": {
    "id": 18,
    "status": "CANCELADO",
    "motivo_cancelamento": "Paciente solicitou cancelamento via WhatsApp",
    "..."
  },
  "mensagem": "Agendamento cancelado com sucesso"
}
```

---

### Convênios

#### Listar Convênios

```
GET /convenios
```

Lista convênios ativos. Opcionalmente filtrados por agenda.

**Permissão:** `convenios.read`

**Parâmetros de Query (opcionais):**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `agenda_id` | integer | Filtra convênios aceitos por esta agenda |

**Exemplo:**
```bash
# Todos os convênios
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/convenios"

# Convênios de uma agenda específica
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/convenios?agenda_id=1"
```

**Resposta:**
```json
{
  "sucesso": true,
  "dados": [
    {
      "id": 1,
      "nome": "Unimed",
      "registro_ans": "123456",
      "tipo": "plano_saude"
    },
    {
      "id": 2,
      "nome": "Particular",
      "registro_ans": null,
      "tipo": "particular"
    }
  ],
  "mensagem": "Convênios listados com sucesso"
}
```

---

## Webhooks

Webhooks permitem receber notificações em tempo real quando eventos ocorrem.

### Configuração

Webhooks são configurados por API Key pelo administrador do sistema (via banco de dados ou painel admin futuro).

### Eventos Disponíveis

| Evento | Quando é disparado |
|--------|--------------------|
| `agendamento.criado` | Novo agendamento criado via API |
| `agendamento.confirmado` | Agendamento confirmado via API |
| `agendamento.cancelado` | Agendamento cancelado via API |

### Formato da Requisição

O sistema faz um `POST` para a URL configurada com:

**Headers:**
```
Content-Type: application/json
X-Webhook-Event: agendamento.criado
X-Webhook-Signature: sha256=HMAC_HASH_AQUI
User-Agent: ClinicaWebhook/1.0
```

**Body:**
```json
{
  "evento": "agendamento.criado",
  "timestamp": "2026-03-03T10:00:00-03:00",
  "dados": {
    "id": 18,
    "numero": "AGD-0018",
    "data": "2026-03-10",
    "hora": "08:00",
    "status": "AGENDADO",
    "paciente": {
      "id": 42,
      "name": "João da Silva",
      "phone": "(84) 99999-1234"
    },
    "agenda": {
      "id": 1,
      "provider": "Dr. João Silva",
      "specialty": "Cardiologia",
      "unit": "Mossoró"
    }
  }
}
```

### Verificação de Assinatura

Para validar que o webhook é legítimo, verifique a assinatura HMAC-SHA256:

```python
import hmac
import hashlib

def verificar_assinatura(payload_body, signature_header, secret):
    expected = 'sha256=' + hmac.new(
        secret.encode('utf-8'),
        payload_body.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature_header)
```

```javascript
const crypto = require('crypto');

function verificarAssinatura(payload, signatureHeader, secret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signatureHeader)
  );
}
```

### Retry e Falhas

- **Tentativas:** 3 (com backoff de 10s, 60s, 300s)
- **Timeout:** 15 segundos por tentativa
- **Desativação automática:** Após 10 falhas consecutivas, o webhook é desativado

---

## Permissões

Cada API Key possui um conjunto de permissões que controla o acesso aos endpoints.

| Permissão | Endpoints |
|-----------|-----------|
| `agendas.read` | `GET /agendas`, `GET /agendas/{id}`, `GET /agendas/{id}/disponibilidade` |
| `pacientes.read` | `GET /pacientes/buscar`, `GET /pacientes/{id}` |
| `pacientes.write` | `POST /pacientes` |
| `agendamentos.read` | `GET /agendamentos/{id}`, `GET /pacientes/{id}/agendamentos` |
| `agendamentos.write` | `POST /agendamentos`, `PUT /agendamentos/{id}/confirmar`, `PUT /agendamentos/{id}/cancelar` |
| `convenios.read` | `GET /convenios` |
| `*` | Acesso total a todos os endpoints |

### Exemplos de configuração:

```bash
# Acesso total
php artisan apikey:manage create --name="Bot WhatsApp" --permissions="*"

# Apenas leitura
php artisan apikey:manage create --name="Bot Consulta" \
  --permissions="agendas.read" --permissions="pacientes.read" --permissions="convenios.read"

# Leitura + agendamento
php artisan apikey:manage create --name="Bot Agendamento" \
  --permissions="agendas.read" --permissions="pacientes.read" --permissions="pacientes.write" \
  --permissions="agendamentos.read" --permissions="agendamentos.write" --permissions="convenios.read"
```

---

## Códigos de Erro

| Código HTTP | Significado |
|-------------|-------------|
| `200` | Sucesso |
| `201` | Recurso criado com sucesso |
| `400` | Requisição inválida |
| `401` | Não autenticado (chave ausente/inválida/expirada) |
| `403` | Sem permissão para o recurso |
| `404` | Recurso não encontrado |
| `422` | Erro de validação (campos inválidos ou regra de negócio) |
| `429` | Rate limit excedido |
| `500` | Erro interno do servidor |

---

## Exemplos de Integração

### Fluxo Completo: Bot de WhatsApp

```
1. Paciente envia mensagem → Bot identifica intenção de agendar
2. Bot busca paciente por CPF/telefone
   GET /pacientes/buscar?telefone=84999991234
3. Se não encontrar, cadastra:
   POST /pacientes { name, phone, birth_date, gender }
4. Bot lista agendas disponíveis:
   GET /agendas?tipo=consulta
5. Bot consulta disponibilidade da agenda escolhida:
   GET /agendas/1/disponibilidade?data=2026-03-10
6. Bot cria o agendamento:
   POST /agendamentos { agenda_id, paciente_id, data_agendamento, hora_agendamento }
7. Sistema dispara webhook → Bot confirma ao paciente
8. No dia anterior, bot confirma:
   PUT /agendamentos/18/confirmar
```

### Python — Exemplo Completo

```python
import requests

API_KEY = "clk_sua_chave_aqui"
BASE_URL = "https://seu-dominio/api/v1/external"
HEADERS = {
    "X-API-Key": API_KEY,
    "Content-Type": "application/json"
}

# 1. Buscar paciente
resp = requests.get(f"{BASE_URL}/pacientes/buscar",
    headers=HEADERS, params={"cpf": "12345678910"})
pacientes = resp.json()["dados"]

if not pacientes:
    # 2. Cadastrar paciente
    resp = requests.post(f"{BASE_URL}/pacientes", headers=HEADERS, json={
        "name": "João da Silva",
        "phone": "(84) 99999-1234",
        "birth_date": "1990-05-20",
        "gender": "M",
        "cpf": "12345678910"
    })
    paciente_id = resp.json()["dados"]["id"]
else:
    paciente_id = pacientes[0]["id"]

# 3. Consultar disponibilidade
resp = requests.get(f"{BASE_URL}/agendas/1/disponibilidade",
    headers=HEADERS, params={"data": "2026-03-10"})
disp = resp.json()["dados"]

# 4. Pegar primeiro horário disponível
horario = next(s["horario"] for s in disp["slots"] if s["status"] == "disponivel")

# 5. Criar agendamento
resp = requests.post(f"{BASE_URL}/agendamentos", headers=HEADERS, json={
    "agenda_id": 1,
    "paciente_id": paciente_id,
    "data_agendamento": "2026-03-10",
    "hora_agendamento": horario
})
agendamento = resp.json()["dados"]
print(f"Agendamento {agendamento['numero']} criado para {agendamento['hora']}")
```

### JavaScript/Node.js — Exemplo

```javascript
const API_KEY = 'clk_sua_chave_aqui';
const BASE_URL = 'https://seu-dominio/api/v1/external';

const headers = {
  'X-API-Key': API_KEY,
  'Content-Type': 'application/json'
};

async function agendarConsulta(cpf, agendaId, data) {
  // Buscar paciente
  const busca = await fetch(`${BASE_URL}/pacientes/buscar?cpf=${cpf}`, { headers });
  const { dados: pacientes } = await busca.json();

  if (!pacientes.length) {
    throw new Error('Paciente não encontrado');
  }

  // Consultar disponibilidade
  const disp = await fetch(
    `${BASE_URL}/agendas/${agendaId}/disponibilidade?data=${data}`, { headers }
  );
  const { dados: disponibilidade } = await disp.json();

  const horario = disponibilidade.slots?.find(s => s.status === 'disponivel');
  if (!horario) throw new Error('Sem horários disponíveis');

  // Criar agendamento
  const resp = await fetch(`${BASE_URL}/agendamentos`, {
    method: 'POST',
    headers,
    body: JSON.stringify({
      agenda_id: agendaId,
      paciente_id: pacientes[0].id,
      data_agendamento: data,
      hora_agendamento: horario.horario
    })
  });

  return await resp.json();
}
```

---

## Gerenciamento de API Keys

```bash
# Criar chave com acesso total
php artisan apikey:manage create --name="Bot WhatsApp" --permissions="*"

# Criar chave com permissões específicas e vinculada a uma unidade
php artisan apikey:manage create --name="Bot Mossoró" \
  --permissions="agendas.read" --permissions="agendamentos.write" \
  --unit-id=1 --rate-limit=120

# Criar chave com data de expiração
php artisan apikey:manage create --name="Parceiro Temp" \
  --permissions="*" --expires=2026-12-31

# Listar todas as chaves
php artisan apikey:manage list

# Revogar uma chave
php artisan apikey:manage revoke --key-id=1
```

---

## Rate Limiting

- Padrão: **60 requisições por minuto** por API Key
- Configurável por chave (parâmetro `--rate-limit` na criação)
- Headers de resposta incluem informações do rate limit:
  - `X-RateLimit-Limit`: Limite total
  - `X-RateLimit-Remaining`: Requisições restantes
  - `Retry-After`: Segundos até reset (quando excedido)

---

## Restrição por Unidade

API Keys podem ser vinculadas a uma unidade específica (parâmetro `--unit-id`). Quando vinculada:
- `GET /agendas` retorna apenas agendas da unidade
- Demais endpoints não são filtrados (pacientes e convênios são globais)

---

## Suporte

Para dúvidas ou problemas com a API, entre em contato com a equipe de TI da clínica.
