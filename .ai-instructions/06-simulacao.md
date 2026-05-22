# 🚀 Passo 6/6: Conteinerização Integrada, Simulação Sob Carga & Documentação Premium

Parabéns pela sua resiliência, jovem Padawan! Ao último passo da nossa jornada de aprendizado você chegou, sim! Suas aplicações (API e Worker) totalmente programadas e validadas por testes robustos localmente já estão.

Agora, o empacotamento completo de produção nós faremos! Criar **Dockerfiles** modernos focados no ecossistema do **`uv`** e atualizar o **Docker Compose** para orquestrar os serviços integrados em uma rede isolada nós devemos! Ao final, simular o comportamento de concorrência com o REST Client e escalar os workers em paralelo para comprovar o Fair Dispatch você fará, completando a sua trilha!

---

## 🐳 1. Criando os Dockerfiles de Produção

Diferente de imagens Python tradicionais que instalam dependências lentas com pip, utilizaremos o gerenciador ultra-rápido `uv` dentro dos nossos containers. Como o arquivo `pyproject.toml` reside na raiz do nosso workspace, utilizaremos o contexto de compilação apontado para a raiz do projeto.

### 📝 Dockerfile da API (`api/Dockerfile`)
Crie o arquivo [api/Dockerfile](file:///api/Dockerfile) de produção:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN pip install uv

COPY pyproject.toml .
RUN uv sync

COPY . .

CMD ["uv", "run", "task", "api-start"]
```

### 📝 Dockerfile do Worker (`worker/Dockerfile`)
Crie o arquivo [worker/Dockerfile](file:///worker/Dockerfile) de produção:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN pip install uv

COPY pyproject.toml .
RUN uv sync

COPY . .

CMD ["uv", "run", "python", "-m", "worker.main"]
```

---

## 🏗️ 2. O Docker Compose Integrado Completo

Com os Dockerfiles estruturados, atualizaremos o arquivo `docker-compose.yml` na raiz do projeto para gerenciar todos os nossos serviços de forma integrada. 

A API e o Worker serão compilados apontando para o contexto global (`.`) e utilizarão o volume compartilhado nomeado **`pedidos_data`** para que o banco SQLite (`pedidos.db`) persistido de forma concorrente em disco seja. Além disso, as diretivas de **Healthcheck** garantirão a resiliência no startup de conexões, aguardando que o container `rabbitmq` esteja saudável antes de inicializar as aplicações.

Substitua o conteúdo do arquivo [docker-compose.yml](file:///docker-compose.yml) pelo blueprint exato da nossa arquitetura de produção:

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq-broker
    ports:
      - "5672:5672"
      - "15672:15672"
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 10s
      timeout: 5s
      retries: 5

  api_produtor:
    build:
      context: .
      dockerfile: api/Dockerfile
    container_name: api-produtor
    ports:
      - "8000:8000"
    environment:
      - RABBITMQ_HOST=rabbitmq
      - RABBITMQ_PORT=5672
      - RABBITMQ_USER=guest
      - RABBITMQ_PASSWORD=guest
      - DB_PATH=/app/data/pedidos.db
    volumes:
      - pedidos_data:/app/data
    depends_on:
      rabbitmq:
        condition: service_healthy

  worker_consumidor:
    build:
      context: .
      dockerfile: worker/Dockerfile
    container_name: worker-consumidor
    environment:
      - RABBITMQ_HOST=rabbitmq
      - RABBITMQ_PORT=5672
      - RABBITMQ_USER=guest
      - RABBITMQ_PASSWORD=guest
      - DB_PATH=/app/data/pedidos.db
    volumes:
      - pedidos_data:/app/data
    depends_on:
      rabbitmq:
        condition: service_healthy

volumes:
  pedidos_data:
```

### 🚀 Compilando e Subindo a Stack Completa
Com a estrutura salva na raiz do workspace, execute o comando para compilar as imagens e iniciar toda a orquestração em background:

```bash
docker compose up --build -d
```

Verifique no console que as imagens do `uv` compiladas rapidamente foram e que os 3 containers executando de forma integrada estão!

---

## 📡 3. Testando via REST Client (`requests.http`)

Para validar o fluxo de concorrência e monitorar o enfileiramento diretamente, criaremos um arquivo `requests.http` na raiz do seu projeto. Ele permite simular chamadas de API de forma nativa na IDE.

Crie o arquivo [requests.http](file:///requests.http) contendo o lote de simulação de carga abaixo:

```http
### Variáveis
@api = http://localhost:8000
@rmq = http://localhost:15672/api
@rmq_auth = Basic guest guest

# =============================================================================
# API
# =============================================================================

### Health
GET {{api}}/health

### Pedido válido (HTTP 202 - Aceito e Enfileirado)
POST {{api}}/pedidos/
Content-Type: application/json

{
  "id": "1",
  "descricao": "Notebook ASUS",
  "valor": 3500.0
}

###

### Pedido com valor inválido (→ HTTP 422 - Rejeitado no Pydantic)
POST {{api}}/pedidos/
Content-Type: application/json

{
  "id": "2",
  "descricao": "Item inválido",
  "valor": -10.0
}

###

### Publicar 3 pedidos em sequência (teste de persistência / fair dispatch)
POST {{api}}/pedidos/
Content-Type: application/json

{
  "id": "3",
  "descricao": "Teclado Mecânico",
  "valor": 450.0
}

###

POST {{api}}/pedidos/
Content-Type: application/json

{
  "id": "4",
  "descricao": "Monitor 4K",
  "valor": 1800.0
}

###

POST {{api}}/pedidos/
Content-Type: application/json

{
  "id": "5",
  "descricao": "Headset Gamer",
  "valor": 320.0
}

# =============================================================================
# RabbitMQ Management API
# =============================================================================

### Visão geral das filas
GET {{rmq}}/queues
Authorization: {{rmq_auth}}

###

### Estado da fila principal
GET {{rmq}}/queues/%2F/pedidos_queue
Authorization: {{rmq_auth}}

###

### Estado da Dead Letter Queue
GET {{rmq}}/queues/%2F/dlx_pedidos
Authorization: {{rmq_auth}}

###

### Exchanges declaradas
GET {{rmq}}/exchanges
Authorization: {{rmq_auth}}

###

### Bindings da fila principal
GET {{rmq}}/bindings/%2F/e/pedidos_exchange/q/pedidos_queue
Authorization: {{rmq_auth}}
```

---

## 📝 4. Documentação Premium (`README.md`)

Crie o arquivo [README.md](file:///README.md) definitivo do seu projeto na raiz do workspace. A documentação premium é o que diferencia engenheiros brilhantes de meros digitadores. O seu README deve conter:
1. **Visão Geral e Arquitetura**: O diagrama de sequência e a explicação do desacoplamento da API e do Worker compartilhando o SQLite.
2. **Requisitos e Setup**: Instruções para instalar o `uv` e sincronizar o ambiente virtual (`uv sync`).
3. **Instruções de Inicialização**: O comando `docker compose up --build -d` para subir a stack conteinerizada.
4. **Simulação Prática**: Como disparar requisições em lote usando o `requests.http` e monitorar logs.
5. **Comportamento de Escala**: Como escalar o processamento paralelo para 3 workers concorrentes e assistir o Fair Dispatch de logs no console:
   ```bash
   docker compose up --scale worker_consumidor=3 -d
   ```
6. **Métricas e Logs**: Comandos para inspecionar a auditoria de logs no container.


---

## 🚀 5. Desafio Extra: Integração Contínua (CI) com GitHub Actions

Na engenharia de software de alta performance, a sua suite de testes deve rodar automaticamente a cada alteração de código! Para consagrar a sua jornada, o seu projeto local para um repositório no **GitHub** você subir deve, e uma pipeline de **Integração Contínua (CI)** com o **GitHub Actions** criar você irá!

O GitHub Actions executará os linters e rodará a suite de testes a cada Push ou Pull Request enviado para o branch principal, garantindo a integridade contínua do seu ecossistema.

### 📝 Criando o Workflow da Pipeline (`.github/workflows/ci.yml`)

Na raiz do seu workspace (`rabbitmq-stack/`), crie a estrutura de pastas `.github/workflows/` se ainda não existir, e crie o arquivo [ci.yml](file:///.github/workflows/ci.yml) contendo a definição ultra-rápida baseada em cache nativo do `uv`:

```yaml
name: Continuous Integration

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

jobs:
  test:
    name: Run Lint and Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
          version: "latest"

      - name: Sync dependencies
        run: uv sync

      - name: Run Ruff Lint
        run: uv run task lint

      - name: Run Test Suite with Coverage Validation
        run: uv run task test-cov
```

### 🛰️ Subindo o Projeto e Ativando a Pipeline

1. **Inicialize o Git localmente** (se ainda não o fez):
   ```bash
   git init
   git branch -M main
   git add .
   git commit -m "feat: implementacao completa do ecossistema rabbitmq-stack"
   ```
2. **Crie um repositório no seu GitHub**:
   * Vá em seu painel do GitHub e crie um novo repositório público ou privado chamado `rabbitmq-stack`.
3. **Associe e suba o código**:
   * Siga as instruções do próprio GitHub para parear o repositório remoto:
     ```bash
     git remote add origin https://github.com/SEU_USUARIO/rabbitmq-stack.git
     git push -u origin main
     ```
4. **Assista a Magia Acontecer**:
   * Acesse a aba **Actions** em seu repositório do GitHub. Você verá a pipeline `Continuous Integration` disparar instantaneamente, instalando o Python, ativando o cache do `uv` em milissegundos, rodando os linters e validando a cobertura de testes em menos de 30 segundos!

---

## 🧙‍♂️ Instruções do Mestre:

A sua jornada de aprendizado do ecossistema assíncrono concluída com glória está, jovem Padawan! Todo o ecossistema integrado rodando na rede conteinerizada apresentar você precisa, sim!

O seu arquivo `README.md` impecável e as chamadas do lote HTTP com logs de concorrência me provar você deve! E para o seu valor na força verdadeiramente atestar, o seu repositório no GitHub com a pipeline do GitHub Actions verde e brilhando no Actions provar você precisa, sim!

> [!IMPORTANT]
> **Fluxo de Aprovação Final e Quiz de Maestria**:
> Primeiro, o ecossistema completo orquestrado, a escala concorrente de workers e a pipeline do GitHub Actions de CI funcionando eu irei avaliar.
> Após atestarmos a integridade e robustez total da sua entrega prática, a você eu aplicarei o **Quiz de Fixação Final Interativo com 4 perguntas estratégicas** sobre concorrência no SQLite3 e o fluxo do Fair Dispatch.
> 
> Com maestria as perguntas responder você deve! Apenas após a sua comprovação teórica, a certificação oficial de conclusão da trilha RabbitMQ Stack outorgada a você será! Que a Força esteja com você nesta consagração final!
