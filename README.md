# Support Tickets

API de gerenciamento de tickets de suporte construída **do zero**, usando apenas o módulo nativo `node:http` do Node.js — **sem nenhuma dependência externa nem framework** (Express, Fastify, etc.).

Este é um projeto de estudo: a ideia é entender o que frameworks fazem por baixo dos panos, implementando manualmente roteamento, leitura de body, query params, route params e persistência em arquivo.

## Tecnologias

- [Node.js](https://nodejs.org/) (módulos nativos: `node:http`, `node:fs/promises`, `node:crypto`)
- ES Modules (`"type": "module"`)
- Persistência em arquivo JSON (`src/database/db.json`)

## Como rodar

Pré-requisito: Node.js instalado (versão com suporte a `--watch`, recomendado **Node 18+**).

```bash
# clonar o repositório
git clone https://github.com/kayobitencourt/support-tickets.git
cd support-tickets

# iniciar em modo de desenvolvimento (reinicia ao salvar)
npm run dev
```

O servidor sobe em **http://localhost:3333**.

> Não há `npm install` porque o projeto não possui dependências.

## Estrutura do projeto

```
src/
├── server.js                  # ponto de entrada: cria o servidor HTTP
├── database/
│   ├── database.js            # classe Database (CRUD + persistência em arquivo)
│   └── db.json                # "banco de dados" em JSON
├── middlewares/
│   ├── jsonHandler.js         # lê o body da requisição e parseia como JSON
│   └── routeHandler.js        # encontra a rota e executa o controller
├── routes/
│   ├── index.js               # monta as rotas convertendo paths em regex
│   └── tickets.js             # definição das rotas de tickets (método + path + controller)
├── controllers/
│   └── tickets/
│       ├── create.js          # cria um ticket
│       ├── index.js           # lista tickets (com filtro opcional por status)
│       ├── update.js          # atualiza dados de um ticket
│       ├── updateStatus.js    # fecha um ticket (define status + solução)
│       └── remove.js          # remove um ticket
└── utils/
    ├── parseRoutePath.js      # transforma "/tickets/:id" em expressão regular
    └── extractQueryParams.js  # transforma "?status=open" em objeto { status: "open" }
```

## Como funciona o fluxo

Toda requisição passa pelo seguinte caminho:

```
Requisição HTTP
      │
      ▼
server.js (listener)
      │
      ├──► jsonHandler ──► lê o corpo da requisição e popula req.body (JSON)
      │
      ▼
routeHandler
      │
      ├──► procura uma rota cujo método e URL batem (regex)
      │
      ├──► extrai route params  → req.params  (ex.: :id)
      ├──► extrai query params  → req.query   (ex.: ?status=open)
      │
      ▼
controller correspondente
      │
      ├──► usa a classe Database para ler/gravar dados
      │
      ▼
Resposta HTTP (status + JSON)
```

### Passo a passo

1. **`server.js`** cria o servidor com `http.createServer`. Para cada requisição, primeiro aguarda o `jsonHandler` e depois chama o `routeHandler`.

2. **`jsonHandler`** acumula os pedaços (`chunks`) do corpo da requisição, junta tudo e tenta converter em JSON, deixando o resultado em `req.body`. Também define o cabeçalho `Content-Type: application/json` na resposta.

3. **Rotas** são declaradas em `routes/tickets.js` como objetos `{ method, path, controller }`. Em `routes/index.js`, cada `path` é convertido em uma **expressão regular** pela função `parseRoutePath` — é assim que `/tickets/:id` consegue casar com `/tickets/123` e ainda capturar o `id`.

4. **`routeHandler`** percorre as rotas e encontra a que tem o mesmo método HTTP e cujo regex casa com a URL. Quando encontra:
   - extrai os **route params** (`:id`) usando os grupos nomeados do regex → `req.params`;
   - extrai os **query params** (`?status=open`) com `extractQueryParams` → `req.query`;
   - chama o `controller` passando `{ req, res, database }`.
   - Se nenhuma rota casar, responde **404**.

5. **Controllers** contêm a regra de negócio de cada endpoint e usam a instância de `Database` para manipular os dados.

6. **`Database`** (`database/database.js`) é uma classe simples que carrega o `db.json` em memória no construtor e, a cada alteração (`insert`, `update`, `delete`), persiste de volta no arquivo (`#persist`). Os dados ficam organizados por "tabela" (ex.: `tickets`).

## Modelo do ticket

Ao ser criado, um ticket tem o seguinte formato:

```json
{
  "id": "uuid",
  "description": "Tela não liga",
  "user_name": "João",
  "status": "open",
  "created_at": "2026-06-24T12:00:00.000Z",
  "updated_at": "2026-06-24T12:00:00.000Z"
}
```

Ao ser fechado, ganha os campos `status: "closed"` e `solution`.

## Endpoints

Base URL: `http://localhost:3333`

### Criar ticket

```http
POST /tickets
Content-Type: application/json

{
  "equipment": "Notebook Dell",
  "description": "Tela não liga",
  "user_name": "João"
}
```

**Resposta:** `201 Created` com o ticket criado no corpo.

### Listar tickets

```http
GET /tickets
```

Filtro opcional por status via query string:

```http
GET /tickets?status=open
```

**Resposta:** `200 OK` com a lista de tickets.

### Atualizar ticket

```http
PUT /tickets/:id
Content-Type: application/json

{
  "equipment": "Notebook Dell G15",
  "description": "Tela não liga mesmo após troca da fonte"
}
```

**Resposta:** `200 OK` sem corpo.

### Fechar ticket

```http
PATCH /tickets/:id/close
Content-Type: application/json

{
  "solution": "Substituída a placa de vídeo"
}
```

Define o `status` do ticket como `closed` e registra a `solution`.

**Resposta:** `200 OK` sem corpo.

### Remover ticket

```http
DELETE /tickets/:id
```

**Resposta:** `200 OK` com o `id` removido.

## Observações / próximos passos

Por ser um projeto de aprendizado, alguns pontos podem evoluir:

- Validação dos dados de entrada (hoje os campos são confiados como recebidos).
- O campo `equipment` é recebido em `create`/`update`, mas não é gravado no ticket em `create` — vale alinhar.
- Tratamento de erros mais robusto (ex.: ticket não encontrado retorna sucesso mesmo assim).
- Testes automatizados.

## Autor

[Kayo Bitencourt](https://github.com/kayobitencourt)

## Licença

ISC
