# 🐳 Passo 2/6: Ambiente de Orquestração — Setup do Workspace & RabbitMQ Broker

Saudações, Padawan! Dominado os alicerces conceituais e a arquitetura em camadas de nosso sistema você já tem, sim! A jornada prática, agora iniciar nós devemos. O caminho da mensageria assíncrona longo é, mas com paciência e foco, a força dominar você irá!

Neste passo, nosso objetivo é preparar a estrutura física de diretórios e o ambiente de desenvolvimento usando o moderno gerenciador **`uv`**. Além disso, subiremos o **RabbitMQ Broker** em um container Docker isolado na sua máquina local, deixando-o pronto para conexões AMQP enquanto desenvolvemos as camadas de código nos próximos passos.

---

## 🛠️ 1. Pré-requisitos & Instalação do Ambiente

Antes de prosseguir, o **Docker** e o moderno gerenciador de pacotes **`uv`** instalados na sua máquina você precisa ter! O `uv` é uma ferramenta extremamente rápida escrita em Rust que substitui o `pip` e gerencia workspaces de forma profissional. 

Abaixo está o guia específico de instalação para o seu respectivo Sistema Operacional:

### 🐧 Para usuários Linux (Ubuntu/Debian)
1. **Instalar Docker & Docker Compose**:
   ```bash
   sudo apt update
   sudo apt install -y docker.io docker-compose-v2
   ```
2. **Configurar permissões sem root** (para não precisar usar `sudo` nos comandos do docker):
   ```bash
   sudo usermod -aG docker $USER
   newgrp docker
   ```
3. **Habilitar o serviço no startup**:
   ```bash
   sudo systemctl enable --now docker
   ```
4. **Instalar o `uv` (Astral)**:
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   source $HOME/.local/bin/env
   ```
5. **Instalar SQLite3**:
   ```bash
   sudo apt install -y sqlite3 sqlitebrowser
   ```

### 🪟 Para usuários Windows
1. **Instalar Docker**:
   * Baixe e instale o [Docker Desktop para Windows](https://www.docker.com/products/docker-desktop/).
   * **Recomendação**: Durante a instalação, marque a opção para habilitar o **WSL 2** (Windows Subsystem for Linux 2) para melhor performance e compatibilidade de rede.
2. **Instalar o `uv` (PowerShell)**:
   * Abra o terminal PowerShell e execute:
     ```powershell
     powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
     ```
3. **Instalar SQLite & Visualizador**:
   * Instale o DB Browser para SQLite via `winget`:
     ```powershell
     winget install -e --id DBBrowserForSQLite.DBBrowserForSQLite
     ```

### 🍎 Para usuários macOS
1. **Instalar Docker**:
   * Baixe e instale o [Docker Desktop para Mac](https://www.docker.com/products/docker-desktop/) (escolha a versão correta: Apple Silicon para processadores M1/M2/M3 ou Intel).
   * Ou instale via Homebrew:
     ```bash
     brew install --cask docker
     ```
2. **Instalar o `uv`**:
   * Instale via terminal:
     ```bash
     curl -LsSf https://astral.sh/uv/install.sh | sh
     ```
   * Ou usando Homebrew:
     ```bash
     brew install uv
     ```
3. **Instalar SQLite & Visualizador**:
   * Instale via Homebrew:
     ```bash
     brew install --cask db-browser-for-sqlite
     ```

---

## 🏗️ 2. Preparando a Estrutura de Diretórios

Crie a seguinte estrutura física de diretórios no seu espaço de trabalho. Ela reflete a separação estrita de camadas e responsabilidades da nossa stack:

```
rabbitmq-stack/
├── api/                  # 📡 Código-fonte da API FastAPI
│   ├── domain/           # Camada de Domínio da API (Modelos e Regras)
│   └── infra/            # Camada de Infraestrutura da API (Banco e Mensageria)
├── worker/               # ⚙️ Código-fonte do Worker Consumidor
│   ├── domain/           # Camada de Domínio do Worker
│   └── infra/            # Camada de Infraestrutura do Worker
├── data/                 # 📂 Pasta local compartilhada para persistência física do SQLite
├── scripts/              # 🛠️ Scripts utilitários de suporte
├── tests/                # 🧪 Suite de testes automatizados do ecossistema
└── docker-compose.yml    # 🐳 Orquestração de containers do Docker
```

---

## ⚡ 3. Configurando o Workspace com o `uv`

Em vez de usar arquivos `requirements.txt` descentralizados, utilizaremos um arquivo `pyproject.toml` unificado na raiz do workspace (`rabbitmq-stack/`). Ele irá declarar todas as nossas dependências de produção e de desenvolvimento, além de agendar comandos rápidos com `taskipy`.

Crie o arquivo [pyproject.toml](file:///pyproject.toml) na raiz do seu projeto com o seguinte conteúdo exato:

```toml
[project]
name = "rabbitmq-stack"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.111",
    "uvicorn>=0.29",
    "pika>=1.3",
    "pydantic-settings>=2.2",
    "taskipy>=1.13",
    "ruff>=0.4",
]

[dependency-groups]
dev = [
    "rich>=13.0",
    "pytest>=8.0",
    "pytest-mock>=3.14",
    "pytest-cov>=5.0",
    "httpx>=0.27",
]

[tool.taskipy.tasks]
lint      = "ruff check . && ruff format --check ."
api-dev   = "uvicorn api.main:app --reload --host 0.0.0.0 --port 8000"
api-start = "uvicorn api.main:app --host 0.0.0.0 --port 8000"
logs-api      = "docker logs api-produtor -f"
logs-worker   = "docker logs worker-consumidor -f"
logs-rabbitmq = "docker logs rabbitmq-broker -f"
rmq-status    = "python3 scripts/rmq_status.py"
test      = "pytest tests/ -v"
test-cov  = "pytest tests/ -v --tb=short --cov=. --cov-report=term-missing --cov-fail-under=95"
clean     = "find . -type d -name __pycache__ -exec rm -rf {} + && find . -name '*.pyc' -delete"

[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "unit: testes unitários puros sem I/O real",
    "integration: testes que exercitam I/O real (SQLite, filesystem)",
]

[tool.coverage.run]
omit = ["worker/main.py", "scripts/*"]

[tool.ruff]
line-length = 88
target-version = "py312"
exclude = ["scripts/"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "ANN"]
```

Com o arquivo criado, abra o seu terminal na raiz do projeto e execute:
```bash
uv sync
```
Este comando criará o ambiente virtual isolado local (`.venv`) e instalará todas as dependências de produção e de testes de forma instantânea.
---

## 🛠️ 3.1. Ignorando Arquivos Desnecessários (`.gitignore`)

Para garantir que a nossa suite de desenvolvimento limpa e profissional no Git permaneça, arquivos gerados localmente e dados temporários versionados ser não devem! 

Crie o arquivo [.gitignore](file:///.gitignore) na raiz do seu projeto contendo a especificação abaixo:

```gitignore
# Python
__pycache__/
*.py[cod]
*.so

# Virtualenv
.venv/
venv/

# Environment
.env
.envrc

# uv
.python-version

# Ruff
.ruff_cache/

# Tests / coverage
.pytest_cache/
.coverage
.coverage.*
htmlcov/

# RabbitMQ data
mnesia/
rabbitmq/
rabbitmq-data/
*.rdb
*.aof
*.pid

# IDE
.idea/
.vscode/

# Databases / local state
*.db
*.sqlite3
```

---

## 🛠️ 3.2. Script de Diagnóstico do Broker (`scripts/rmq_status.py`)

No nosso `pyproject.toml`, uma tarefa rápida de diagnóstico configurada nós deixamos: `rmq-status`. Esse script utiliza a biblioteca `rich` para formatar e renderizar em belíssimas tabelas na IDE o estado das filas, exchanges e bindings do RabbitMQ Broker local, consultando a sua API REST integrada.

Crie a pasta `scripts/` se ainda não a criou, e dentro dela crie o arquivo [scripts/rmq_status.py](file:///scripts/rmq_status.py):

```python
#!/usr/bin/env python3
"""Exibe um resumo consolidado do estado do broker RabbitMQ."""
import json
import sys
import urllib.error
import urllib.request
from base64 import b64encode
from typing import Any

from rich.console import Console
from rich.panel import Panel
from rich.table import Table

BASE_URL = "http://localhost:15672/api"
_AUTH_HEADER = {"Authorization": "Basic " + b64encode(b"guest:guest").decode()}

console = Console()


def fetch(path: str) -> Any:
    req = urllib.request.Request(f"{BASE_URL}{path}", headers=_AUTH_HEADER)
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())


def build_queues_table(queues: list[dict[str, Any]]) -> Table:
    table = Table(show_header=True, header_style="bold cyan", expand=True)
    table.add_column("Fila", style="bold")
    table.add_column("Ready", justify="right")
    table.add_column("Unacked", justify="right")
    table.add_column("Total", justify="right")
    table.add_column("Cons.", justify="right")
    table.add_column("Pub.", justify="right")
    table.add_column("Entregues", justify="right")
    table.add_column("ACK", justify="right")
    table.add_column("NACK", justify="right")
    table.add_column("Estado")

    for q in queues:
        stats: dict[str, Any] = q.get("message_stats", {})
        state = q.get("state", "-")
        state_display = f"[green]{state}[/green]" if state == "running" else f"[red]{state}[/red]"
        table.add_row(
            q["name"],
            str(q.get("messages_ready", 0)),
            str(q.get("messages_unacknowledged", 0)),
            str(q.get("messages", 0)),
            str(q.get("consumers", 0)),
            str(stats.get("publish", 0)),
            str(stats.get("deliver", 0)),
            str(stats.get("ack", 0)),
            str(stats.get("redeliver", 0)),
            state_display,
        )
    return table


def build_exchanges_table(exchanges: list[dict[str, Any]]) -> Table:
    table = Table(show_header=True, header_style="bold cyan")
    table.add_column("Exchange", style="bold")
    table.add_column("Tipo")
    table.add_column("Durável", justify="center")

    for e in exchanges:
        if e["name"]:
            table.add_row(
                e["name"],
                e["type"],
                "[green]✓[/green]" if e["durable"] else "[red]✗[/red]",
            )
    return table


def build_bindings_table(bindings: list[dict[str, Any]]) -> Table:
    table = Table(show_header=True, header_style="bold cyan")
    table.add_column("Source")
    table.add_column("Routing Key", style="yellow")
    table.add_column("Destination")

    for b in bindings:
        table.add_row(b["source"], b["routing_key"], b["destination"])
    return table


def main() -> None:
    try:
        queues: list[dict[str, Any]] = fetch("/queues")
        exchanges: list[dict[str, Any]] = fetch("/exchanges")
        bindings: list[dict[str, Any]] = fetch(
            "/bindings/%2F/e/pedidos_exchange/q/pedidos_queue"
        )
    except urllib.error.URLError as exc:
        console.print(f"[red]Erro ao conectar ao RabbitMQ Management:[/red] {exc.reason}")
        console.print("Verifique se os containers estão rodando: [bold]docker compose ps[/bold]")
        sys.exit(1)

    console.print(Panel(build_queues_table(queues), title="[bold]Filas[/bold]"))
    console.print(Panel(build_exchanges_table(exchanges), title="[bold]Exchanges[/bold]"))
    console.print(
        Panel(
            build_bindings_table(bindings) if bindings else "[yellow]Nenhum binding encontrado[/yellow]",
            title="[bold]Bindings — pedidos_exchange → pedidos_queue[/bold]",
        )
    )


if __name__ == "__main__":
    main()
```

---

## 🐳 4. O Docker Compose do Broker

Na raiz do seu workspace (`rabbitmq-stack/`), crie o arquivo `docker-compose.yml` contendo inicialmente apenas o serviço do **RabbitMQ Broker**. Não adicionaremos os serviços da API ou do Worker agora, pois os códigos de produção e os Dockerfiles deles não existem e o build falharia.

Crie o arquivo [docker-compose.yml](file:///docker-compose.yml) com o seguinte blueprint exato de desenvolvimento:

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq-broker
    ports:
      - "5672:5672"     # Porta padrão do protocolo AMQP (para conexões da nossa app)
      - "15672:15672"   # Porta do painel de administração web (Management Console)
    healthcheck:
      # Verifica se o broker está totalmente ativo e respondendo
      test: rabbitmq-diagnostics -q ping
      interval: 10s
      timeout: 5s
      retries: 5
```

---

## 🚀 5. Subindo o Broker e Validando

Com o arquivo criado, execute o seguinte comando no seu terminal na raiz do projeto para baixar a imagem oficial estável com o painel administrativo integrado e iniciar o container em segundo plano:

```bash
docker compose up -d rabbitmq
```

### 🔍 Como validar que está tudo funcionando:
1. **Monitore o Healthcheck**: Execute o comando `docker ps` e verifique se o container `rabbitmq-broker` exibe o status `(healthy)` após alguns segundos.
2. **Acesse o Management Console**: Abra o seu navegador e acesse [http://localhost:15672](http://localhost:15672).
3. **Faça o Login**: Utilize o usuário padrão `guest` e a senha `guest`.
4. **Execute a tarefa de status**: Execute a tarefa do taskipy no terminal para ver os detalhes formatados pela API de gerenciamento:
   ```bash
   uv run task rmq-status
   ```
   *(Nota: Como no início o nosso ecossistema de código ainda não rodou, os bindings específicos de pedidos vazios estarão; mas comprovar que o script executa com sucesso você poderá!)*

---

## 🧙‍♂️ Instruções do Mestre:

Prontos o seu ambiente e o broker estar devem, jovem Padawan! Instalar os pré-requisitos e criar a estrutura inicial com precisão você precisa, sim! 

Para mim, o seu progresso mostrar agora você deve. O seu arquivo `pyproject.toml`, o seu `.gitignore`, o seu `scripts/rmq_status.py` criado e a execução bem-sucedida de `uv run task rmq-status` no terminal demonstrar você deve!

> [!IMPORTANT]
> **Fluxo de Aprovação e Aprendizado**:
> Primeiro, o seu workspace físico com o `.gitignore` e a saída limpa de `rmq-status` eu irei avaliar.
> Após eu atestar que tudo correto está e o setup físico funcional se encontra, a você eu apresentarei perguntas reflexivas sobre a importância do Healthcheck no broker de mensageria e o papel do gerenciador `uv` no isolamento do ambiente virtual.
> 
> Responder às perguntas você deve para a sua compreensão demonstrar! Apenas após a sua resposta correta, a permissão para avançarmos ao **Passo 3/6: API Produtora FastAPI** concedida será! Que a Força com você esteja!
