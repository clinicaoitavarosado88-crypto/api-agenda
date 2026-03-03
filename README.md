# API Externa - Clínica Oitava Rosado

API REST para integração de sistemas externos (bots de WhatsApp, IA, parceiros) com os módulos de agendamentos, valores de procedimentos, cartão de desconto e ordens de serviço da clínica.

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
  - [Valores e Cartão de Desconto](#valores-e-cartão-de-desconto)
  - [Ordens de Serviço](#ordens-de-serviço)
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

### Valores e Cartão de Desconto

#### Consultar Valores de Procedimentos

```
GET /procedimentos/valores
```

Retorna os valores dos procedimentos da **tabela de negociação** do convênio particular. Os valores já vêm definidos por forma de pagamento (dinheiro, pix, cartão débito, cartão à vista, cartão parcelado).

A resposta possui duas seções:
- **`valores_sem_plano`** — valores para pacientes **SEM** Cartão de Desconto
- **`cartao_desconto.valores`** — valores para pacientes **COM** Cartão de Desconto (já definidos na tabela, sem cálculo)

Se CPF informado, verifica se o paciente tem Cartão de Desconto ativo e se o plano é aceito para o procedimento.

> **Importante:** Use o `convenio_id` do convênio **Particular** da unidade desejada. Valores de plano de saúde **NÃO** devem ser consultados por este endpoint.

**Permissão:** `valores.read`

**Fonte dos valores:**
1. `convenio_procedimentos` (tabela de negociação) — campo `payment_structure` com valores por forma de pagamento nas seções `sem_plano` e `planos`
2. `Procedure.value` — fallback quando o procedimento não está na tabela de negociação

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `procedure_ids[]` | integer[] | Sim | IDs dos procedimentos (máx 50) |
| `convenio_id` | integer | Sim | ID do convênio (particular da unidade) |
| `cpf` | string | Não | CPF do paciente para verificar cartão de desconto |

**Exemplo — Somente valores negociados:**
```bash
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/procedimentos/valores?procedure_ids[]=10&procedure_ids[]=15&convenio_id=2"
```

**Resposta:**
```json
{
  "sucesso": true,
  "dados": {
    "procedimentos": [
      {
        "id": 10,
        "nome": "Ultrassonografia Abdominal",
        "codigo_tuss": "40901017",
        "valor_negociado": 150.00,
        "origem_valor": "tabela_negociacao",
        "valores_sem_plano": {
          "dinheiro": 150.00,
          "pix": 150.00,
          "cartao_debito": 160.00,
          "cartao_avista": 160.00,
          "cartao_parcelado": 170.00
        },
        "cartao_desconto": null
      },
      {
        "id": 15,
        "nome": "Hemograma Completo",
        "codigo_tuss": "40304361",
        "valor_negociado": 25.00,
        "origem_valor": "tabela_negociacao",
        "valores_sem_plano": {
          "dinheiro": 25.00,
          "pix": 25.00,
          "cartao_debito": 25.00,
          "cartao_avista": 25.00,
          "cartao_parcelado": 25.00
        },
        "cartao_desconto": null
      }
    ]
  },
  "mensagem": "Valores consultados com sucesso"
}
```

**Exemplo — Com cartão de desconto:**
```bash
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/procedimentos/valores?procedure_ids[]=10&convenio_id=2&cpf=12345678910"
```

**Resposta (paciente com cartão ativo e plano aceito):**
```json
{
  "sucesso": true,
  "dados": {
    "procedimentos": [
      {
        "id": 10,
        "nome": "Ultrassonografia Abdominal",
        "codigo_tuss": "40901017",
        "valor_negociado": 150.00,
        "origem_valor": "tabela_negociacao",
        "valores_sem_plano": {
          "dinheiro": 150.00,
          "pix": 150.00,
          "cartao_debito": 160.00,
          "cartao_avista": 160.00,
          "cartao_parcelado": 170.00
        },
        "cartao_desconto": {
          "tem_cartao": true,
          "status": "adimplente",
          "plano": "Individual",
          "plano_aceito": true,
          "valores": {
            "dinheiro": 100.00,
            "pix": 100.00,
            "cartao_debito": 110.00,
            "cartao_avista": 115.00,
            "cartao_parcelado": 120.00
          }
        }
      }
    ]
  },
  "mensagem": "Valores consultados com sucesso"
}
```

> **Nota:** Os valores em `cartao_desconto.valores` são pré-definidos na tabela de negociação (seção "Planos"), **NÃO** são calculados a partir de percentuais. Eles são cadastrados pelo administrador junto com os valores `sem_plano`.

**Resposta (paciente com cartão, mas plano NÃO aceito para o procedimento):**
```json
{
  "sucesso": true,
  "dados": {
    "procedimentos": [
      {
        "id": 10,
        "nome": "Ultrassonografia Abdominal",
        "codigo_tuss": "40901017",
        "valor_negociado": 150.00,
        "origem_valor": "tabela_negociacao",
        "valores_sem_plano": {
          "dinheiro": 150.00,
          "pix": 150.00,
          "cartao_debito": 160.00,
          "cartao_avista": 160.00,
          "cartao_parcelado": 170.00
        },
        "cartao_desconto": {
          "tem_cartao": true,
          "status": "adimplente",
          "plano": "Individual",
          "plano_aceito": false,
          "valores": null,
          "mensagem": "Plano do cartão não aceito para este procedimento"
        }
      }
    ]
  },
  "mensagem": "Valores consultados com sucesso"
}
```

**Resposta (paciente sem cartão):**
```json
{
  "sucesso": true,
  "dados": {
    "procedimentos": [
      {
        "id": 10,
        "nome": "Ultrassonografia Abdominal",
        "codigo_tuss": "40901017",
        "valor_negociado": 150.00,
        "origem_valor": "tabela_negociacao",
        "valores_sem_plano": {
          "dinheiro": 150.00,
          "pix": 150.00,
          "cartao_debito": 160.00,
          "cartao_avista": 160.00,
          "cartao_parcelado": 170.00
        },
        "cartao_desconto": {
          "tem_cartao": false,
          "status": "nao_encontrado",
          "plano": null,
          "plano_aceito": false,
          "valores": null,
          "mensagem": "Paciente não possui Cartão de Desconto"
        }
      }
    ]
  },
  "mensagem": "Valores consultados com sucesso"
}
```

**Campo `origem_valor`:**

| Valor | Significado |
|-------|-------------|
| `tabela_negociacao` | Valor encontrado na tabela de negociação do convênio |
| `tabela_procedimento` | Fallback: valor padrão do procedimento (sem tabela de negociação) |

---

#### Verificar Cartão de Desconto

```
GET /cartao-desconto/verificar
```

Verifica o status do cartão de desconto de um paciente por CPF. Consulta base local e API externa (cartaooitavarosado.com.br).

**Permissão:** `valores.read`

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `cpf` | string | Sim | CPF do paciente |

**Exemplo:**
```bash
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/cartao-desconto/verificar?cpf=12345678910"
```

**Resposta (paciente com cartão ativo):**
```json
{
  "sucesso": true,
  "dados": {
    "tem_cartao": true,
    "status": "adimplente",
    "plano": "Individual",
    "pode_usar_cartao": true,
    "tipo_vinculo": "titular",
    "mensagem": "Paciente adimplente. Pode usar Cartão de Desconto."
  },
  "mensagem": "Verificação realizada"
}
```

> **Nota:** Este endpoint apenas verifica o **status** do cartão. Os **valores com desconto** são retornados pelo endpoint `GET /procedimentos/valores` (na seção `cartao_desconto.valores`), pois os valores variam por procedimento e já estão definidos na tabela de negociação.

**Resposta (paciente inadimplente):**
```json
{
  "sucesso": true,
  "dados": {
    "tem_cartao": true,
    "status": "inadimplente",
    "plano": "Individual",
    "pode_usar_cartao": false,
    "tipo_vinculo": "titular",
    "mensagem": "Paciente inadimplente. Regularizar situação para usar Cartão."
  },
  "mensagem": "Verificação realizada"
}
```

**Resposta (sem cartão):**
```json
{
  "sucesso": true,
  "dados": {
    "tem_cartao": false,
    "status": "nao_encontrado",
    "plano": null,
    "pode_usar_cartao": false,
    "tipo_vinculo": null,
    "mensagem": "Paciente não possui Cartão de Desconto"
  },
  "mensagem": "Verificação realizada"
}
```

---

### Ordens de Serviço

#### Criar Ordem de Serviço

```
POST /ordens-servico
```

Cria uma nova Ordem de Serviço. Auto-gera número da OS, senha de chamada, login e senha para consulta de resultados. Dispara webhook `ordem_servico.criada`.

**Permissão:** `ordens_servico.write`

**Body (JSON):**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `unit_id` | integer | Sim | ID da unidade |
| `paciente_id` | integer | Sim | ID do paciente |
| `convenio_id` | integer | Sim | ID do convênio |
| `tipo_atendimento` | string | Sim | `consulta`, `exame`, `procedimento` ou `retorno` |
| `procedimentos` | array | Sim | Lista de procedimentos (mín 1) |
| `procedimentos[].procedimento_id` | integer | Sim | ID do procedimento |
| `procedimentos[].quantidade` | number | Não | Quantidade (padrão: 1) |
| `procedimentos[].valor_unitario` | number | Não | Valor unitário (padrão: valor da tabela) |
| `procedimentos[].descricao` | string | Não | Descrição (padrão: nome do procedimento) |
| `agendamento_id` | integer | Não | ID do agendamento a vincular |
| `provider_id` | integer | Não | ID do profissional |
| `prioridade` | string | Não | `normal` (padrão), `preferencial` ou `urgente` |
| `observacoes` | string | Não | Observações (máx 1000 chars) |
| `numero_carteira` | string | Não | Número da carteira do convênio |
| `validade_carteira` | date | Não | Validade da carteira |

**Exemplo:**
```bash
curl -X POST -H "X-API-Key: clk_..." \
  -H "Content-Type: application/json" \
  -d '{
    "unit_id": 1,
    "paciente_id": 42,
    "convenio_id": 1,
    "tipo_atendimento": "exame",
    "agendamento_id": 18,
    "procedimentos": [
      { "procedimento_id": 10, "quantidade": 1, "valor_unitario": 25.00 }
    ],
    "observacoes": "Criado via WhatsApp"
  }' \
  "https://seu-dominio/api/v1/external/ordens-servico"
```

**Resposta (201):**
```json
{
  "sucesso": true,
  "dados": {
    "id": 100,
    "numero_os": "OS202603030001",
    "status": "aguardando",
    "tipo_atendimento": "exame",
    "prioridade": "normal",
    "data_os": "2026-03-03",
    "hora_os": "10:30:00",
    "senha_chamada": "N001",
    "login_resultado": "OS202603030001",
    "senha_resultado": "482917",
    "valor_total": 25.00,
    "observacoes": "Criado via WhatsApp",
    "paciente": {
      "id": 42,
      "name": "João da Silva",
      "cpf": "98765432100",
      "phone": "(84) 99999-1234"
    },
    "convenio": { "id": 1, "nome": "Particular" },
    "unidade": { "id": 1, "name": "Mossoró" },
    "procedimentos": [
      {
        "id": 201,
        "procedure_id": 10,
        "descricao": "Hemograma Completo",
        "quantidade": 1.0,
        "valor_unitario": 25.00,
        "valor_total": 25.00
      }
    ],
    "agendamento_id": 18
  },
  "mensagem": "Ordem de Serviço criada com sucesso"
}
```

**Erros possíveis (422):**

| Mensagem | Causa |
|----------|-------|
| `Agendamento não pertence a este paciente` | paciente_id diverge do agendamento |
| `Este agendamento já possui uma Ordem de Serviço vinculada` | OS já criada para o agendamento |

---

#### Ver Ordem de Serviço

```
GET /ordens-servico/{id}
```

Retorna dados completos de uma Ordem de Serviço, incluindo paciente, convênio, procedimentos e status do atendimento.

**Permissão:** `ordens_servico.read`

**Exemplo:**
```bash
curl -H "X-API-Key: clk_..." \
  "https://seu-dominio/api/v1/external/ordens-servico/100"
```

**Resposta:**
```json
{
  "sucesso": true,
  "dados": {
    "id": 100,
    "numero_os": "OS202603030001",
    "status": "aguardando",
    "tipo_atendimento": "exame",
    "prioridade": "normal",
    "data_os": "2026-03-03",
    "hora_os": "10:30:00",
    "senha_chamada": "N001",
    "login_resultado": "OS202603030001",
    "senha_resultado": "482917",
    "valor_total": 25.00,
    "valor_desconto": 0.00,
    "observacoes": "Criado via WhatsApp",
    "data_hora_chegada": null,
    "data_hora_chamada": null,
    "data_hora_inicio": null,
    "data_hora_fim": null,
    "paciente": {
      "id": 42,
      "name": "João da Silva",
      "cpf": "98765432100",
      "phone": "(84) 99999-1234"
    },
    "convenio": { "id": 1, "nome": "Particular" },
    "unidade": { "id": 1, "name": "Mossoró" },
    "profissional": null,
    "procedimentos": [
      {
        "id": 201,
        "procedure_id": 10,
        "descricao": "Hemograma Completo",
        "quantidade": 1.0,
        "valor_unitario": 25.00,
        "valor_total": 25.00,
        "tipo_item": "procedimento",
        "procedimento": {
          "id": 10,
          "nome": "Hemograma Completo",
          "codigo_tuss": "40304361"
        }
      }
    ]
  },
  "mensagem": "Ordem de Serviço encontrada"
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
| `ordem_servico.criada` | Nova Ordem de Serviço criada via API |

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
| `valores.read` | `GET /procedimentos/valores`, `GET /cartao-desconto/verificar` |
| `ordens_servico.read` | `GET /ordens-servico/{id}` |
| `ordens_servico.write` | `POST /ordens-servico` |
| `*` | Acesso total a todos os endpoints |

### Exemplos de configuração:

```bash
# Acesso total
php artisan apikey:manage create --name="Bot WhatsApp" --permissions="*"

# Apenas leitura (agendas, pacientes, convênios, valores)
php artisan apikey:manage create --name="Bot Consulta" \
  --permissions="agendas.read" --permissions="pacientes.read" \
  --permissions="convenios.read" --permissions="valores.read"

# Leitura + agendamento + valores
php artisan apikey:manage create --name="Bot Agendamento" \
  --permissions="agendas.read" --permissions="pacientes.read" --permissions="pacientes.write" \
  --permissions="agendamentos.read" --permissions="agendamentos.write" \
  --permissions="convenios.read" --permissions="valores.read"

# Fluxo completo: agendamento + OS + valores
php artisan apikey:manage create --name="Bot Completo" \
  --permissions="agendas.read" --permissions="pacientes.read" --permissions="pacientes.write" \
  --permissions="agendamentos.read" --permissions="agendamentos.write" \
  --permissions="convenios.read" --permissions="valores.read" \
  --permissions="ordens_servico.read" --permissions="ordens_servico.write"
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
2. Bot busca paciente por CPF/telefone:
   GET /pacientes/buscar?cpf=12345678910
3. Se não encontrar, cadastra:
   POST /pacientes { name, phone, birth_date, gender, cpf }
4. Bot lista agendas disponíveis:
   GET /agendas?tipo=consulta
5. Bot consulta valores do procedimento (tabela negociação + cartão desconto):
   GET /procedimentos/valores?procedure_ids[]=10&convenio_id=2&cpf=12345678910
6. Bot mostra preços ao paciente e consulta disponibilidade:
   GET /agendas/1/disponibilidade?data=2026-03-10
7. Bot cria o agendamento:
   POST /agendamentos { agenda_id, paciente_id, data_agendamento, hora_agendamento }
8. Sistema dispara webhook agendamento.criado → Bot confirma ao paciente
9. No dia anterior, bot confirma presença:
   PUT /agendamentos/18/confirmar
10. No dia do atendimento, bot/sistema cria a Ordem de Serviço:
    POST /ordens-servico { unit_id, paciente_id, convenio_id, procedimentos, agendamento_id }
11. Sistema dispara webhook ordem_servico.criada → Bot informa senha e dados ao paciente
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

CONVENIO_PARTICULAR_ID = 2  # ID do convênio Particular da unidade

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

# 3. Consultar valores dos procedimentos (tabela negociação + cartão desconto)
resp = requests.get(f"{BASE_URL}/procedimentos/valores", headers=HEADERS, params={
    "procedure_ids[]": [10, 15],
    "convenio_id": CONVENIO_PARTICULAR_ID,
    "cpf": "12345678910"
})
valores = resp.json()["dados"]["procedimentos"]
for proc in valores:
    valor_pix = proc["valores_sem_plano"]["pix"]
    print(f"{proc['nome']}: R$ {valor_pix} (PIX sem plano)")
    cartao = proc["cartao_desconto"]
    if cartao and cartao.get("plano_aceito") and cartao.get("valores"):
        valor_pix_cartao = cartao["valores"]["pix"]
        print(f"  Com cartão (PIX): R$ {valor_pix_cartao}")

# 4. Consultar disponibilidade
resp = requests.get(f"{BASE_URL}/agendas/1/disponibilidade",
    headers=HEADERS, params={"data": "2026-03-10"})
disp = resp.json()["dados"]

# 5. Pegar primeiro horário disponível
horario = next(s["horario"] for s in disp["slots"] if s["status"] == "disponivel")

# 6. Criar agendamento
resp = requests.post(f"{BASE_URL}/agendamentos", headers=HEADERS, json={
    "agenda_id": 1,
    "paciente_id": paciente_id,
    "data_agendamento": "2026-03-10",
    "hora_agendamento": horario
})
agendamento = resp.json()["dados"]
print(f"Agendamento {agendamento['numero']} criado para {agendamento['hora']}")

# 7. No dia do atendimento: criar Ordem de Serviço
resp = requests.post(f"{BASE_URL}/ordens-servico", headers=HEADERS, json={
    "unit_id": 1,
    "paciente_id": paciente_id,
    "convenio_id": CONVENIO_PARTICULAR_ID,
    "tipo_atendimento": "exame",
    "agendamento_id": agendamento["id"],
    "procedimentos": [
        {"procedimento_id": 10, "quantidade": 1, "valor_unitario": 90.00}
    ],
    "observacoes": "Criado via WhatsApp"
})
os = resp.json()["dados"]
print(f"OS {os['numero_os']} criada — Senha: {os['senha_chamada']}")
print(f"Resultado: login={os['login_resultado']} senha={os['senha_resultado']}")
```

### JavaScript/Node.js — Exemplo

```javascript
const API_KEY = 'clk_sua_chave_aqui';
const BASE_URL = 'https://seu-dominio/api/v1/external';
const CONVENIO_PARTICULAR_ID = 2;

const headers = {
  'X-API-Key': API_KEY,
  'Content-Type': 'application/json'
};

// Consultar valores com cartão de desconto
async function consultarValores(procedureIds, cpf) {
  const params = new URLSearchParams({ convenio_id: CONVENIO_PARTICULAR_ID });
  procedureIds.forEach(id => params.append('procedure_ids[]', id));
  if (cpf) params.append('cpf', cpf);

  const resp = await fetch(`${BASE_URL}/procedimentos/valores?${params}`, { headers });
  const { dados } = await resp.json();
  return dados.procedimentos;
}

// Fluxo completo: agendar + criar OS
async function agendarECriarOS(cpf, agendaId, data, procedureIds) {
  // Buscar paciente
  const busca = await fetch(`${BASE_URL}/pacientes/buscar?cpf=${cpf}`, { headers });
  const { dados: pacientes } = await busca.json();
  if (!pacientes.length) throw new Error('Paciente não encontrado');
  const pacienteId = pacientes[0].id;

  // Consultar valores
  const valores = await consultarValores(procedureIds, cpf);

  // Consultar disponibilidade
  const disp = await fetch(
    `${BASE_URL}/agendas/${agendaId}/disponibilidade?data=${data}`, { headers }
  );
  const { dados: disponibilidade } = await disp.json();
  const horario = disponibilidade.slots?.find(s => s.status === 'disponivel');
  if (!horario) throw new Error('Sem horários disponíveis');

  // Criar agendamento
  const agResp = await fetch(`${BASE_URL}/agendamentos`, {
    method: 'POST', headers,
    body: JSON.stringify({
      agenda_id: agendaId,
      paciente_id: pacienteId,
      data_agendamento: data,
      hora_agendamento: horario.horario
    })
  });
  const agendamento = (await agResp.json()).dados;

  // Criar Ordem de Serviço
  const osResp = await fetch(`${BASE_URL}/ordens-servico`, {
    method: 'POST', headers,
    body: JSON.stringify({
      unit_id: 1,
      paciente_id: pacienteId,
      convenio_id: CONVENIO_PARTICULAR_ID,
      tipo_atendimento: 'exame',
      agendamento_id: agendamento.id,
      procedimentos: valores.map(v => ({
        procedimento_id: v.id,
        quantidade: 1,
        valor_unitario: v.valor_negociado
      }))
    })
  });

  return { agendamento, os: (await osResp.json()).dados, valores };
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
- `GET /ordens-servico/{id}` retorna apenas OS da unidade
- `POST /ordens-servico` só permite criar OS na unidade da chave
- Demais endpoints não são filtrados (pacientes e convênios são globais)

---

## Suporte

Para dúvidas ou problemas com a API, entre em contato com a equipe de TI da clínica.
