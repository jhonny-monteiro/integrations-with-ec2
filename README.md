# API Go — Gerenciamento de Alunos

API REST desenvolvida em Go para gerenciamento de alunos, com integração a banco de dados PostgreSQL, containerização via Docker e pipelines de CI/CD completos no GitHub Actions para entrega em AWS EC2 e AWS ECS.

---

## Tecnologias utilizadas

| Tecnologia | Versão | Função |
|---|---|---|
| Go | 1.15+ | Linguagem principal |
| Gin | v1.7.7 | Framework HTTP |
| GORM | v1.22.4 | ORM |
| PostgreSQL | latest | Banco de dados |
| Docker | — | Containerização |
| GitHub Actions | — | CI/CD |
| AWS EC2 | — | Deploy direto do binário |
| AWS ECS | — | Deploy via container |

---

## Estrutura do projeto

```
.
├── main.go                  # Ponto de entrada da aplicação
├── main_test.go             # Testes de integração
├── go.mod / go.sum          # Dependências Go
├── Dockerfile               # Imagem Docker da aplicação
├── docker-compose.yml       # PostgreSQL + pgAdmin para desenvolvimento local
├── controllers/
│   └── controller.go        # Handlers das rotas HTTP
├── database/
│   └── db.go                # Conexão e migração do banco de dados
├── models/
│   └── alunos.go            # Model Aluno e validações
├── routes/
│   └── route.go             # Definição das rotas
├── templates/
│   ├── index.html           # Página de listagem de alunos
│   └── 404.html             # Página de erro 404
├── assets/
│   ├── index.css
│   └── 404.css
└── .github/
    └── workflows/
        ├── go.yml           # CI: testes + build + disparo dos workflows seguintes
        ├── docker.yml       # CD: build e push de imagem para Docker Hub
        ├── EC2.yml          # CD: deploy do binário em instância EC2
        ├── ECS.yml          # CD: deploy do container no AWS ECS com rollback
        └── LoadTest.yml     # Teste de carga via AWS
```

---

## Endpoints da API

### Alunos

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/alunos` | Lista todos os alunos |
| `GET` | `/alunos/:id` | Busca aluno por ID |
| `GET` | `/alunos/cpf/:cpf` | Busca aluno por CPF |
| `POST` | `/alunos` | Cria um novo aluno |
| `PATCH` | `/alunos/:id` | Atualiza dados de um aluno |
| `DELETE` | `/alunos/:id` | Remove um aluno |

### Outras rotas

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/index` | Página HTML com listagem de alunos |
| `GET` | `/:nome` | Rota de saudação personalizada |

### Exemplo de body (POST/PATCH)

```json
{
  "nome": "João Silva",
  "rg": "123456789",
  "cpf": "12345678901"
}
```

### Regras de validação do model `Aluno`

| Campo | Regra |
|---|---|
| `nome` | Obrigatório, não vazio |
| `rg` | Exatamente 9 dígitos numéricos |
| `cpf` | Exatamente 11 dígitos numéricos |

---

## Configuração por variáveis de ambiente

A aplicação lê todas as configurações via variáveis de ambiente, sem valores hardcoded.

| Variável | Descrição | Exemplo |
|---|---|---|
| `HOST` | Host do banco de dados | `localhost` |
| `DB_USER` | Usuário do banco | `root` |
| `PASSWORD` | Senha do banco | `root` |
| `DBNAME` | Nome do banco | `root` |
| `DBPORT` | Porta do banco | `5432` |
| `PORT` | Porta da aplicação (padrão: `8000`) | `8000` |

---

## Executando localmente

### Pré-requisitos

- Go 1.15+
- Docker e Docker Compose

### 1. Subir o banco de dados

```bash
docker compose up -d
```

Isso sobe o PostgreSQL na porta `5432` e o pgAdmin na porta `54321`.

**pgAdmin:** acesse `http://localhost:54321`
- Email: `gui@alura.com`
- Senha: `123456`

### 2. Exportar variáveis de ambiente

```bash
export HOST=localhost
export DB_USER=root
export PASSWORD=root
export DBNAME=root
export DBPORT=5432
export PORT=8000
```

### 3. Rodar a aplicação

```bash
go run main.go
```

A API estará disponível em `http://localhost:8000`.

### 4. Executar os testes

```bash
go test -v main_test.go
```

> Os testes requerem o banco de dados ativo. Certifique-se de que o `docker compose` está rodando.

---

## Docker

### Build manual da imagem

O `Dockerfile` utiliza a imagem `ubuntu:latest` e copia apenas o binário compilado (`main`) e os templates HTML.

```bash
# Compilar o binário
go build -o main .

# Build da imagem
docker build -t go_ci .

# Executar o container
docker run -p 8000:8000 \
  -e HOST=<db_host> \
  -e DB_USER=<user> \
  -e PASSWORD=<password> \
  -e DBNAME=<dbname> \
  -e DBPORT=5432 \
  go_ci
```

A imagem publicada no Docker Hub encontra-se em: `jhonnymonteiro/go_ci:<run_number>`

---

## Pipelines CI/CD — GitHub Actions

O fluxo de CI/CD é dividido em workflows encadeados, todos disparados a partir do workflow principal `go.yml`.

### Visão geral do fluxo

```
push / pull_request
        │
        ▼
   [ go.yml ]
   ├── job: test   → Testes em matriz (Go 1.17, 1.18 / Ubuntu)
   ├── job: build  → Compila o binário e salva como artefato
   └── job: docker → Chama docker.yml
                          │
                          └── (implicitamente, os workflows de deploy
                               EC2.yml e ECS.yml são chamados
                               como workflow_call)
```

---

### `go.yml` — Integração Contínua

**Disparo:** todo `push` ou `pull_request` em qualquer branch.

| Job | Descrição |
|---|---|
| `test` | Roda `go test` em matriz de versões Go (1.17, 1.18) e sistemas (ubuntu-latest, ubuntu-22.04). Sobe o banco via `docker compose`. |
| `build` | Compila o binário (`go build -o main .`) e faz upload como artefato `program`. Depende de `test`. |
| `docker` | Chama o workflow reutilizável `docker.yml`. Depende de `build`. |

**Secrets necessárias:** nenhuma neste workflow (as secrets são passadas via `secrets: inherit` para os filhos).

---

### `docker.yml` — Build e Push no Docker Hub

**Disparo:** `workflow_call` (chamado por `go.yml`).

| Passo | Ação |
|---|---|
| Setup Buildx | Configura o Docker Buildx para builds multi-plataforma |
| Download artefato | Baixa o binário `program` gerado no job `build` |
| Login Docker Hub | Autentica com `jhonnymonteiro` usando secret |
| Build e Push | Constrói a imagem a partir do `Dockerfile` e faz push com a tag `jhonnymonteiro/go_ci:<run_number>` |

**Secrets necessárias:**

| Secret | Descrição |
|---|---|
| `PASSWORD_DOCKER_HUB` | Senha ou token de acesso do Docker Hub |

---

### `EC2.yml` — Deploy direto em instância EC2

**Disparo:** `workflow_call`.

| Passo | Ação |
|---|---|
| Download artefato | Baixa o binário compilado |
| SSH Deploy | Copia o binário para `/home/<REMOTE_USER>` via SSH (exclui `postgres-data`) |
| Execução remota | Para o processo anterior (`pkill main`), dá permissão de execução e inicia a aplicação em background com `nohup` |

**Secrets necessárias:**

| Secret | Descrição |
|---|---|
| `SSH_PRIVATE_KEY` | Chave privada SSH para acesso à instância |
| `REMOTE_HOST` | IP ou DNS público da instância EC2 |
| `REMOTE_USER` | Usuário SSH (ex.: `ubuntu`, `ec2-user`) |
| `DBHOST` | Host do banco de dados na AWS |
| `DBUSER` | Usuário do banco |
| `DBPASSWORD` | Senha do banco |
| `DBNAME` | Nome do banco |
| `DBPORT` | Porta do banco |

---

### `ECS.yml` — Deploy em AWS ECS com rollback automático

**Disparo:** `workflow_call`.

| Passo | Ação |
|---|---|
| Configurar credenciais AWS | Configura `aws-access-key-id` e `aws-secret-access-key` para a região `us-east-2` |
| Obter task definition | Busca a task definition `Tarefa_API-GO` via AWS CLI e salva como `task-definition.json` |
| Backup | Copia a task definition atual como `task-definition.json.old` para possível rollback |
| Atualizar imagem | Substitui a imagem na task definition pela nova (`jhonnymonteiro/go_ci:<run_number>`) e injeta as variáveis de ambiente |
| Deploy ECS | Atualiza o serviço `Servico_API-Go` no cluster `API-Go` e aguarda estabilização |
| Health check | Aguarda 30s e faz requisição HTTP ao Load Balancer. Se falhar, aciona rollback |
| Rollback | Em caso de falha no health check, faz deploy da task definition antiga automaticamente |

**Cluster e serviço AWS:**
- Cluster: `API-Go`
- Serviço: `Servico_API-Go`
- Task definition: `Tarefa_API-GO`
- Container: `Go`
- Load Balancer: `LB-API-Go-363554340.us-east-2.elb.amazonaws.com:8000`

**Secrets necessárias:**

| Secret | Descrição |
|---|---|
| `ID_CHAVE_ACESSO` | AWS Access Key ID |
| `CHAVE_SECRETA` | AWS Secret Access Key |
| `DBHOST` | Host do banco de dados |
| `DBUSER` | Usuário do banco |
| `DBPASSWORD` | Senha do banco |
| `DBNAME` | Nome do banco |
| `DBPORT` | Porta do banco |

---

### `LoadTest.yml` — Teste de carga

**Disparo:** `workflow_call`.

Configura credenciais AWS na região `us-east-2` como base para execução de testes de carga contra a infraestrutura na AWS.

**Secrets necessárias:**

| Secret | Descrição |
|---|---|
| `ID_CHAVE_ACESSO` | AWS Access Key ID |
| `CHAVE_SECRETA` | AWS Secret Access Key |

---

## Testes

Os testes cobrem os principais handlers da API utilizando `httptest` e `testify/assert`. São testes de integração que se comunicam com o banco de dados real.

| Teste | O que valida |
|---|---|
| `TestVerificaStatusCodeDaSaudacaoComParametro` | Status 200 e body correto na rota `/:nome` |
| `TestListaTodosOsAlunosHanlder` | Status 200 na listagem de alunos |
| `TestBucaAlunoPorCPFHandler` | Status 200 na busca por CPF |
| `TestBuscaAlunoPorIDHandler` | Status 200, nome, CPF e RG corretos na busca por ID |
| `TestDeletaAlunoHandler` | Status 200 ao deletar aluno |
| `TestEditaUmAlunoHandler` | Status 200 e dados atualizados após edição |

Cada teste que acessa o banco utiliza um aluno mock criado via `CriaAlunoMock()` e removido via `DeletaAlunoMock()` com `defer`.
