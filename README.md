# Caixa da Associação de Moradores

**Cloud Challenge · Desafio 07** · Cloud Computing · Senac · Prof. Roni Maciel · 2026.2

**Grupo XX** *(substituir XX pelo número do grupo)*

| Integrante | Papel |
|---|---|
| Caio Sabino | DEV (back-end) |
| Marcos Vinicius | DADOS |
| Arthur Vininicus | NUVEM |
| Thauan Bezerra | QA |
| Pedro Rodrigues | DEV (interface) |

- **Nível de implantação:** A / B / C *(preencher o nível alcançado)*
- **URL pública (Nível A):** *(preencher)*
- **Vídeo do pitch (90 s):** *(colar o link)*

---

## 1. Problema e usuário

A associação de moradores presta contas todo mês em assembleia, mas as despesas ficam num caderno e os totais são somados na calculadora, então todo mês "não fecha" por alguns centavos. O usuário é o(a) **tesoureiro(a)**, que precisa registrar despesas e emitir totais **exatos** por categoria e período, prontos para a prestação de contas.

O produto registra despesas num banco de dados na nuvem, calcula os totais **no próprio banco** com tipo decimal exato e impede qualquer alteração em despesas já conferidas.

---

## 2. Requisitos da ficha e status

| Código | Requisito | Status |
|---|---|---|
| RF1 | Registrar despesa: descrição, valor (> 0, até 2 casas decimais), categoria e data (não futura) | ⬜ Pendente |
| RF2 | Listar despesas por período (de/até) e categoria | ⬜ Pendente |
| RF3 | Totais por categoria e total geral do período, calculados no banco e exatos ao centavo **(regra central)** | ⬜ Pendente |
| RF4 | Marcar despesa como conferida; conferida não pode ser editada nem excluída | ⬜ Pendente |
| RF5 | Editar descrição, valor e categoria de despesa não conferida | ⬜ Pendente |
| Geral | `GET /health` testando a conexão com o banco | ⬜ Pendente |
| Geral | Validação no servidor com erros 400, 404, 409 e 503 sem expor detalhes internos | ⬜ Pendente |
| Geral | Consultas parametrizadas e `DATABASE_URL` fora do código | ⬜ Pendente |

*Legenda: ✅ Concluído · 🟡 Parcial · ⬜ Pendente. Atualizar antes das 21h30.*

---

## 3. Arquitetura

```mermaid
flowchart LR
    U[Navegador<br/>tesoureiro] -->|HTTPS| A[Azure App Service<br/>Node.js 20 + Express]
    A -->|TLS, porta 5432<br/>DATABASE_URL| B[(Azure Database for PostgreSQL<br/>Servidor Flexível)]
    A -. logs .-> L[Fluxo de log]
```

**Caminho de uma requisição:** o navegador chama a API no App Service; o Express valida a entrada, abre uma conexão do pool (`pg`) usando a string de `DATABASE_URL` (variável de ambiente do App Service) e executa uma consulta parametrizada no PostgreSQL; o resultado volta como JSON.

### Decisão justificada: `NUMERIC(10,2)` no banco, somando no SQL

- **O que se ganha:** exatidão ao centavo. Em ponto flutuante (`FLOAT`), `0.1 + 0.2` resulta em `0.30000000000000004`, e erros assim acumulam nas somas mensais, que é exatamente o problema do cliente. `NUMERIC` guarda o valor decimal exato. Somar no banco (`SUM ... GROUP BY`) evita converter o valor para `number` do JavaScript e trafega só os totais.
- **O que se perde:** a lógica de totais fica acoplada ao SQL, e `NUMERIC` é um pouco mais lento que `FLOAT` (irrelevante neste volume). O driver `pg` devolve `NUMERIC` como **texto**, então a API mantém os valores como string.

### Outras decisões

| Decisão | Escolha | Justificativa |
|---|---|---|
| Banco | PostgreSQL Servidor Flexível | `NUMERIC` nativo e `CHECK`; serviço gerenciado (backup e manutenção pela Azure) |
| Categorias | `CHECK IN (...)` | Cinco categorias fixas; mais simples que tabela própria, mas menos flexível *(confirmar)* |
| Exclusão | Física, só para despesas não conferidas | Conferida é bloqueada com 409 *(confirmar)* |
| Fuso do "hoje" | *(Brasília ou UTC: definir e documentar)* | O App Service roda em UTC; depois das 21h em Brasília já é "amanhã" em UTC |
| Valor na API | Ponto decimal (`12.50`) | A vírgula é tratada só na interface |
| Categoria com acento | *(`manutencao` sem acento ou `manutenção`: definir)* | Evita problemas de codificação |

---

## 4. Modelo de dados

```sql
CREATE TABLE despesas (
  id            SERIAL PRIMARY KEY,
  descricao     VARCHAR(200)  NOT NULL CHECK (length(trim(descricao)) > 0),
  valor         NUMERIC(10,2) NOT NULL CHECK (valor > 0),
  categoria     VARCHAR(20)   NOT NULL
                CHECK (categoria IN ('manutencao','limpeza','eventos','administrativo','outros')),
  data_despesa  DATE          NOT NULL,
  conferida     BOOLEAN       NOT NULL DEFAULT false,
  criada_em     TIMESTAMPTZ   NOT NULL DEFAULT now()
);
```

| Coluna | Tipo | Restrição | Motivo |
|---|---|---|---|
| `id` | `SERIAL` | PRIMARY KEY | Identificador |
| `descricao` | `VARCHAR(200)` | NOT NULL, não vazia | Toda despesa precisa de descrição |
| `valor` | `NUMERIC(10,2)` | NOT NULL, `> 0` | Dinheiro exato, 2 casas |
| `categoria` | `VARCHAR(20)` | NOT NULL, `CHECK IN (...)` | Apenas as 5 categorias da ficha |
| `data_despesa` | `DATE` | NOT NULL | Data da despesa (a regra "não futura" é validada na API) |
| `conferida` | `BOOLEAN` | NOT NULL, padrão `false` | Bloqueia edição e exclusão |
| `criada_em` | `TIMESTAMPTZ` | padrão `now()` | Auditoria, com fuso |

O script completo está em [`banco/schema.sql`](banco/schema.sql).

---

## 5. API

| Método | Rota | Descrição | Respostas |
|---|---|---|---|
| GET | `/health` | Testa a conexão com o banco; devolve `iniciado_em` | 200, 503 |
| POST | `/api/despesas` | Cadastra despesa (RF1) | 201, 400 |
| GET | `/api/despesas?de=&ate=&categoria=` | Lista com filtros (RF2) | 200, 400 |
| GET | `/api/despesas/totais?de=&ate=` | Totais por categoria e total geral (RF3) | 200, 400 |
| PUT | `/api/despesas/:id` | Edita descrição, valor e categoria (RF5) | 200, 400, 404, 409 |
| PATCH | `/api/despesas/:id/conferir` | Marca como conferida (RF4) | 200, 404, 409 |
| DELETE | `/api/despesas/:id` | Exclui despesa não conferida | 204, 404, 409 |

Requisições prontas em [`requests.http`](requests.http) (extensão *REST Client* do VS Code).

---

## 6. Como executar

### Pré-requisitos

Node.js 20+, Git e acesso a um PostgreSQL 16 (Azure ou local).

### Variáveis de ambiente

Copie `.env.example` para `.env` (o `.env` **nunca** vai para o Git):

```
DATABASE_URL=postgres://usuario:senha@servidor.postgres.database.azure.com:5432/banco
PORT=3000
```

Se a senha tiver caracteres especiais (`@`, `#`, `/`, `:`), codifique-os na URL (por exemplo `@` vira `%40`).

### Banco de dados

```bash
psql "$DATABASE_URL" -f banco/schema.sql
psql "$DATABASE_URL" -f banco/dados-iniciais.sql   # opcional, dados fictícios
```

### Aplicação

```bash
npm install
npm start        # ou: npm run dev
```

Abra `http://localhost:3000/health` para conferir a conexão com o banco.

### Implantação na Azure

1. Grupo de recursos `rg-cloud-gXX`, com as tags `aula = cloud-challenge` e `grupo = gXX`.
2. App Service Linux, Node 20 LTS, plano Gratuito F1.
3. Azure Database for PostgreSQL Servidor Flexível (v16, Burstable B1ms), acesso público, permitir serviços do Azure e adicionar o IP atual.
4. Em Configurações do App Service, adicionar `DATABASE_URL` com a string de conexão.
5. Publicar com `az webapp up` ou pela extensão do VS Code, e conferir `/health` e o Fluxo de log.

---

## 7. Testes

Resultados registrados com data e hora em [`evidencias/`](evidencias/).

| # | Caso | Tipo | Entrada | Esperado | Obtido | Resultado |
|---|---|---|---|---|---|---|
| 1 | CA1: valor zero | Negativo | `valor = 0` | 400 com mensagem | | |
| 2 | CA1: valor negativo | Negativo | `valor = -5` | 400 | | |
| 3 | CA1: 3 casas decimais | Negativo | `valor = 12.345` | 400 | | |
| 4 | CA2: data futura | Negativo | `data` amanhã | 400 | | |
| 5 | CA3: soma exata | Positivo | 10,10 + 20,20 + 0,70 na mesma categoria | Total **31.00** | | |
| 6 | CA4: período filtrado | Positivo | `de`/`ate` com despesas dentro e fora | Só as dentro, limites inclusivos | | |
| 7 | CA5: editar conferida | Negativo | `PUT` em despesa conferida | 409 | | |
| 8 | Id inexistente | Negativo | `PUT`/`DELETE` em id que não existe | 404 | | |
| 9 | Banco indisponível | Negativo | `/health` sem conexão | 503 sem detalhes internos | | |

### Prova de persistência

Registro com a palavra do dia: `GXX-PALAVRA` *(preencher)*. Registro do professor (Nível A): `PROF-HHMM` *(preencher)*.

| Print | Conteúdo | Arquivo |
|---|---|---|
| 1 | Registro listado pela aplicação | `evidencias/` |
| 2 | `iniciado_em` do `/health` antes do reinício | `evidencias/` |
| 3 | Após o reinício: `iniciado_em` novo e o registro continua listado | `evidencias/` |
| 4 | Mesma linha em consulta direta ao banco | `evidencias/` |

### Configuração segura (E5)

Print das variáveis de ambiente com o valor oculto e do `.gitignore` com `.env`, em `evidencias/`.

---

## 8. Segurança

- `DATABASE_URL` só em variável de ambiente; `.env` no `.gitignore`.
- Consultas **parametrizadas** (`$1`, `$2`), sem concatenar texto do usuário no SQL.
- Usuário de banco da aplicação **sem privilégio de administrador**.
- Validação no servidor; mensagens de erro sem detalhes internos (o detalhe técnico fica só no log).
- Somente dados fictícios.

---

## 9. Dificuldade superada

*(Preencher durante a noite, no formato abaixo.)*

- **Problema:**
- **Diagnóstico:**
- **Causa:**
- **Solução:**

---

## 10. Limitações conhecidas e próximos passos

**Limitações**
- Sem autenticação: qualquer pessoa com a URL acessa o sistema.
- A conferência não registra quem conferiu.
- Sem paginação na listagem.
- *(acrescentar outras limitações encontradas)*

**Próximos passos**
- Login do tesoureiro e registro de quem conferiu cada despesa.
- Orçamento mensal por categoria com alerta ao ultrapassar.
- Exportação dos totais em CSV.
- Comprovante de cada despesa no Blob Storage.

---

## 11. Estrutura do repositório

```
07-caixa-moradores/
├── README.md
├── .gitignore            (inclui .env)
├── .env.example          (nomes das variáveis, com valores falsos)
├── package.json
├── server.js
├── db.js
├── requests.http
├── banco/
│   ├── schema.sql
│   └── dados-iniciais.sql
├── evidencias/           (prints com data e hora visíveis)
└── public/               (interface web)
```
