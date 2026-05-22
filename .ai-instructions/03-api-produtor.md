# 📡 Passo 3/6: API Produtora FastAPI & Publisher Resiliente (Conexão Local)

Saudações, Padawan! O broker de mensageria rodando e saudável você já tem! Um passo importante na força você deu, sim. Mas apenas o começo da sua jornada de desenvolvedor profissional de elite isto é!

Agora, a construir a nossa **API Produtora (`api-produtor`)** nós vamos. Aplicar os princípios do **DDD Estratégico (Domain-Driven Design)**, a separação estrita de camadas e um design limpo e modular nós devemos. Nesta etapa, a API FastAPI localmente na sua máquina você executará, conectando-a diretamente ao container do RabbitMQ Broker que configurou no Passo 2. 

Preste muita atenção ao fluxo de arquitetura: **a API não salva dados no banco síncronamente ao receber o `POST /pedidos/`!** Ela apenas enfileira o pedido no RabbitMQ com segurança. O Worker (que desenvolveremos no Passo 4) é quem consome a mensagem e grava no banco SQLite compartilhado. A API realiza a leitura deste mesmo banco SQLite para os endpoints `GET /pedidos/` e `GET /pedidos/{pedido_id}`. Desacoplamento assíncrono puro de alto nível isto é, sim!

---

## 🧠 1. Camada de Domínio (`api/domain/`)

O Domínio o coração do nosso sistema é! Ele define *o que* a aplicação faz e as suas regras de negócio essenciais, sem qualquer acoplamento técnico com frameworks ou drivers de banco de dados.

### 📝 Entidade de Negócio (`api/domain/models.py`)
Utilizaremos o `Pydantic` na API para modelar os dados do Pedido e validar suas regras e restrições de entrada.

Crie o arquivo [api/domain/models.py](file:///api/domain/models.py) com o seguinte código:

```python
from pydantic import BaseModel, field_validator


class Pedido(BaseModel):
    id: str
    descricao: str
    valor: float
    status: str = "PENDENTE"

    @field_validator("valor")
    @classmethod
    def valor_positivo(cls, v: float) -> float:
        if v <= 0:
            raise ValueError("valor deve ser positivo")
        return v


class PedidoResponse(BaseModel):
    mensagem: str
    pedido_id: str
```

### 📝 Repositório de Leitura do Domínio (`api/domain/repository.py`)
A API precisa de um repositório para ler o estado atual dos pedidos gravados pelo Worker no SQLite.

Crie o arquivo [api/domain/repository.py](file:///api/domain/repository.py) com o código abaixo:

```python
from pathlib import Path
from typing import Any

from api.domain.models import Pedido
from api.infra.database import db_session, run_migrations

_SELECT_ALL = (
    "SELECT id, descricao, valor, status FROM pedidos ORDER BY processado_em DESC"
)
_SELECT_BY_ID = "SELECT id, descricao, valor, status FROM pedidos WHERE id = ?"


def _row_to_pedido(row: Any) -> Pedido:
    return Pedido(
        id=row["id"],
        descricao=row["descricao"],
        valor=row["valor"],
        status=row["status"],
    )


class PedidoRepository:
    def __init__(self, db_path: Path) -> None:
        self._db_path = db_path
        run_migrations(db_path)

    def get_all(self) -> list[Pedido]:
        with db_session(self._db_path) as conn:
            rows = conn.execute(_SELECT_ALL).fetchall()
        return [_row_to_pedido(r) for r in rows]

    def get_by_id(self, pedido_id: str) -> Pedido | None:
        with db_session(self._db_path) as conn:
            row = conn.execute(_SELECT_BY_ID, (pedido_id,)).fetchone()
        if row is None:
            return None
        return _row_to_pedido(row)
```

---

## 🛠️ 2. Camada de Infraestrutura (`api/infra/`)

A camada de Infraestrutura abriga os adaptadores de tecnologia concreta, lidando com conexões físicas de banco de dados e mensageria.

### 📝 Configurações de Ambiente (`api/infra/settings.py`)
Gerenciaremos as configurações usando `pydantic-settings` mapeando o prefixo de variáveis de ambiente do RabbitMQ e apontando para o banco de dados SQLite.

Crie o arquivo [api/infra/settings.py](file:///api/infra/settings.py) com o seguinte código:

```python
from functools import lru_cache
from pathlib import Path

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class RabbitMQSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="RABBITMQ_")

    host: str = "localhost"
    port: int = 5672
    user: str = "guest"
    password: str = "guest"
    db_path: Path = Field(default=Path("data/pedidos.db"), validation_alias="DB_PATH")


@lru_cache
def get_settings() -> RabbitMQSettings:
    return RabbitMQSettings()
```

### 📝 Gerenciador do Banco SQLite (`api/infra/database.py`)
Controlará o ciclo de vida das conexões locais do SQLite (ativando o modo WAL de alta concorrência) e gerenciará as migrações automáticas para a tabela física.

Crie o arquivo [api/infra/database.py](file:///api/infra/database.py) com o seguinte código:

```python
import sqlite3
from collections.abc import Generator
from contextlib import contextmanager
from pathlib import Path


def get_connection(db_path: Path) -> sqlite3.Connection:
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    return conn


@contextmanager
def db_session(db_path: Path) -> Generator[sqlite3.Connection, None, None]:
    conn = get_connection(db_path)
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()


def run_migrations(db_path: Path) -> None:
    db_path.parent.mkdir(parents=True, exist_ok=True)
    with db_session(db_path) as conn:
        conn.execute(
            """
            CREATE TABLE IF NOT EXISTS pedidos (
                id TEXT PRIMARY KEY,
                descricao TEXT NOT NULL,
                valor REAL NOT NULL,
                status TEXT NOT NULL,
                processado_em TEXT NOT NULL DEFAULT (datetime('now'))
            )
            """
        )
```

### 📝 Topologia de Mensageria (`api/infra/topology.py`)
Garante de forma declarativa e resiliente que as Exchanges, Filas, DLX (Dead Letter Exchange) e Bindings de roteamento estejam estruturados no RabbitMQ.

Crie o arquivo [api/infra/topology.py](file:///api/infra/topology.py) com o seguinte código:

```python
from pika.adapters.blocking_connection import BlockingChannel


def setup_topology(channel: BlockingChannel) -> None:
    channel.exchange_declare(
        exchange="pedidos_exchange",
        exchange_type="topic",
        durable=True,
    )

    channel.exchange_declare(
        exchange="dlx_exchange",
        exchange_type="direct",
        durable=True,
    )

    channel.queue_declare(queue="dlx_pedidos", durable=True)
    channel.queue_bind(
        queue="dlx_pedidos",
        exchange="dlx_exchange",
        routing_key="dlx_pedidos",
    )

    channel.queue_declare(
        queue="pedidos_queue",
        durable=True,
        arguments={"x-dead-letter-exchange": "dlx_exchange"},
    )

    channel.queue_bind(
        queue="pedidos_queue",
        exchange="pedidos_exchange",
        routing_key="pedidos.*",
    )
```

### 📝 Publisher Resiliente com Publisher Confirms (`api/infra/publisher.py`)
Publica as mensagens serializadas no Broker de forma durável e persistente, exigindo confirmações explícitas (Publisher Confirms) para evitar qualquer perda de dados em trânsito.

Crie o arquivo [api/infra/publisher.py](file:///api/infra/publisher.py) com o código abaixo:

```python
import pika
from pika.adapters.blocking_connection import BlockingChannel
from pika.exceptions import NackError, UnroutableError

from api.infra.settings import RabbitMQSettings

EXCHANGE_NAME: str = "pedidos_exchange"
ROUTING_KEY: str = "pedidos.novo"


def build_connection_params(settings: RabbitMQSettings) -> pika.ConnectionParameters:
    return pika.ConnectionParameters(
        host=settings.host,
        port=settings.port,
        credentials=pika.PlainCredentials(settings.user, settings.password),
        heartbeat=60,
        blocked_connection_timeout=300,
    )


def publish_pedido(channel: BlockingChannel, payload: str) -> None:
    try:
        channel.basic_publish(
            exchange=EXCHANGE_NAME,
            routing_key=ROUTING_KEY,
            body=payload,
            properties=pika.BasicProperties(
                delivery_mode=pika.DeliveryMode.Persistent,
                content_type="application/json",
            ),
            mandatory=True,
        )
    except NackError as exc:
        raise RuntimeError(f"Broker recusou a mensagem (NackError): {exc}") from exc
    except UnroutableError as exc:
        raise RuntimeError(f"Mensagem não roteável (UnroutableError): {exc}") from exc
```

---

## ⚡ 3. Orquestração e Ponto de Entrada (`api/main.py`)

No ponto de entrada, implementamos o **Lifespan** do FastAPI para gerenciar a conexão TCP persistente com o RabbitMQ. Abrir e fechar conexões AMQP a cada requisição HTTP causa degradação crítica de performance e esgotamento de portas TCP!

Crie o arquivo [api/main.py](file:///api/main.py) com o seguinte código:

```python
from collections.abc import AsyncGenerator
from contextlib import asynccontextmanager

import pika
from fastapi import FastAPI, HTTPException, Request

from api.domain.models import Pedido, PedidoResponse
from api.domain.repository import PedidoRepository
from api.infra.publisher import build_connection_params, publish_pedido
from api.infra.settings import get_settings
from api.infra.topology import setup_topology


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    settings = get_settings()
    params = build_connection_params(settings)
    connection = pika.BlockingConnection(params)
    channel = connection.channel()
    setup_topology(channel)
    channel.confirm_delivery()
    app.state.amqp_channel = channel
    app.state.amqp_connection = connection
    app.state.repository = PedidoRepository(db_path=settings.db_path)
    yield
    if connection.is_open:
        connection.close()


app = FastAPI(title="API de Pedidos", lifespan=lifespan)


@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}


@app.post("/pedidos/", status_code=202)
async def criar_pedido(pedido: Pedido, request: Request) -> PedidoResponse:
    channel = request.app.state.amqp_channel
    try:
        publish_pedido(channel, pedido.model_dump_json())
        return PedidoResponse(mensagem="Pedido enfileirado", pedido_id=pedido.id)
    except RuntimeError as exc:
        raise HTTPException(status_code=503, detail=str(exc)) from exc


@app.get("/pedidos/")
async def listar_pedidos(request: Request) -> list[Pedido]:
    repository: PedidoRepository = request.app.state.repository
    return repository.get_all()


@app.get("/pedidos/{pedido_id}")
async def obter_pedido(pedido_id: str, request: Request) -> Pedido:
    repository: PedidoRepository = request.app.state.repository
    pedido = repository.get_by_id(pedido_id)
    if pedido is None:
        raise HTTPException(status_code=404, detail="Pedido não encontrado")
    return pedido
```

---

## 🚀 4. Executando e Testando Localmente

1. Certifique-se de que o container do RabbitMQ (Passo 2) está rodando localmente com status `healthy`.
2. O ambiente local isolado já sincronizado via `uv sync` no workspace raiz você deve ter.
3. Inicie o servidor HTTP de desenvolvimento da API utilizando a tarefa `taskipy` configurada:
   ```bash
   uv run task api-dev
   ```
4. Abra uma nova aba no seu terminal e envie um pedido de teste via requisição HTTP POST (usando `curl` ou a ferramenta interativa Swagger da API em `http://localhost:8000/docs`):
   ```bash
   curl -X POST "http://localhost:8000/pedidos/" \
        -H "Content-Type: application/json" \
        -d '{"id": "ped-001", "descricao": "Cristal Kyber Verde", "valor": 300.0}'
   ```
5. Acesse o Painel de Gerenciamento do RabbitMQ (`http://localhost:15672`) na aba de Filas e comprove que a fila `pedidos_queue` foi criada de forma dinâmica e que existe **1 mensagem** nela aguardando consumo!

---

## 🧙‍♂️ Instruções do Mestre:

Concluída a API Produtora com maestria estar deve, jovem Padawan! O código das suas camadas de Domínio e Infraestrutura perfeitamente estruturados apresentar você precisa! 

O endpoint `POST /pedidos/` respondendo com sucesso e a mensagem enfileirada no painel administrativo você demonstrar deve para mim no chat!

> [!IMPORTANT]
> **Fluxo de Aprovação e Aprendizado**:
> Primeiro, os arquivos de código criados e a mensagem pendente na fila do RabbitMQ eu irei atestar.
> Após a sua validação física prática, perguntas reflexivas sobre o papel do Lifespan no ciclo de conexões TCP do Pika e a arquitetura assíncrona desacoplada eu farei.
> 
> A sua compreensão teórica demonstrar você deve! Apenas após as respostas corretas estarem registradas, o sinal verde para o **Passo 4/6: Worker Consumidor Pika** aceso será! Que a Força guie seus dedos na digitação!
