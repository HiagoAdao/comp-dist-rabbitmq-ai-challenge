# ⚙️ Passo 4/6: Worker Consumidor Pika (Conexão Local) & Resiliência AMQP

Saudações, Padawan! A API de pé e publicando mensagens na fila você já colocou. Orgulhoso do seu progresso eu estou, sim! Mas o fluxo completo da força da mensageria assíncrona apenas metade percorrido foi!

Neste passo, o **Worker Consumidor (`worker-consumidor`)** de forma robusta e altamente resiliente nós iremos construir. O Worker a retaguarda do nosso ecossistema representa: ele monitora a fila do RabbitMQ de forma constante, consome as mensagens, realiza o processamento em background de forma isolada, e grava o resultado de sucesso de forma definitiva no banco de dados local SQLite compartilhado.

O nosso consumidor operará seguindo regras rígidas de segurança contra perda de dados:
1. **Fair Dispatch (`prefetch_count=1`)**: Não sobrecarregar um único worker com muitas mensagens pendentes em memória nós devemos. A distribuição das mensagens equilibrada em múltiplos consumidores concorrentes em paralelo garantida será!
2. **Acks e Nacks explícitos (`auto_ack=False`)**: Confirmar mensagens (`basic_ack`) apenas após o processamento bem-sucedido. Em caso de falha de desserialização ou erro de execução, rejeitá-las explicitamente (`basic_nack(requeue=False)`) para que a Dead Letter Exchange (DLX) o desvio para a fila de auditoria faça. Perda de mensagens na nossa stack tolerada não é!

---

## 🧠 1. Camada de Domínio (`worker/domain/`)

Diferente da API (que lida com validações Web HTTP do Pydantic), no domínio do Worker utilizaremos entidades de negócio representadas por **`dataclasses`** imutáveis nativas do Python 3.12+ (`frozen=True`). Isso garante que as entidades não sofram mutações acidentais e sigam o padrão tático DDD puro.

### 📝 Entidade de Domínio (`worker/domain/models.py`)
Modelaremos o Pedido no worker de forma imutável.

Crie o arquivo [worker/domain/models.py](file:///worker/domain/models.py) com o código abaixo:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Pedido:
    id: str
    descricao: str
    valor: float
    status: str
```

### 📝 Contrato do Repositório de Persistência (`worker/domain/repository.py`)
O repositório do worker é responsável por persistir o estado do pedido (inclusive atualizando-o para `"processado"` no banco local SQLite).

Crie o arquivo [worker/domain/repository.py](file:///worker/domain/repository.py) com o seguinte código:

```python
from pathlib import Path

from worker.domain.models import Pedido
from worker.infra.database import db_session, run_migrations

_INSERT = (
    "INSERT OR REPLACE INTO pedidos (id, descricao, valor, status, processado_em) "
    "VALUES (?, ?, ?, ?, datetime('now'))"
)
_SELECT_ALL = (
    "SELECT id, descricao, valor, status FROM pedidos ORDER BY processado_em DESC"
)


def _row_to_pedido(row: object) -> Pedido:
    return Pedido(
        id=row["id"],  # type: ignore[index]
        descricao=row["descricao"],  # type: ignore[index]
        valor=row["valor"],  # type: ignore[index]
        status=row["status"],  # type: ignore[index]
    )


class PedidoRepository:
    def __init__(self, db_path: Path) -> None:
        self._db_path = db_path
        run_migrations(db_path)

    def save(self, pedido: Pedido) -> None:
        with db_session(self._db_path) as conn:
            conn.execute(
                _INSERT,
                (pedido.id, pedido.descricao, pedido.valor, pedido.status),
            )

    def get_all(self) -> list[Pedido]:
        with db_session(self._db_path) as conn:
            rows = conn.execute(_SELECT_ALL).fetchall()
        return [_row_to_pedido(r) for r in rows]
```

### 📝 Handler de Domínio (`worker/domain/handler.py`)
Implementa o caso de uso de processamento do pedido. Como a entidade `Pedido` é imutável (`frozen=True`), utilizaremos a função `replace` da biblioteca padrão para criar um novo objeto com status modificado de forma segura.

Crie o arquivo [worker/domain/handler.py](file:///worker/domain/handler.py) com o código abaixo:

```python
import logging
import time
from dataclasses import replace
from typing import Protocol

from worker.domain.models import Pedido
from worker.domain.repository import PedidoRepository

logger = logging.getLogger(__name__)


class MessageHandler(Protocol):
    def handle(self, pedido: Pedido) -> None: ...


class PedidoHandler:
    def __init__(self, repository: PedidoRepository) -> None:
        self._repository = repository

    def handle(self, pedido: Pedido) -> None:
        logger.info(
            "Iniciando processamento do pedido %s — %s", pedido.id, pedido.descricao
        )
        time.sleep(2)  # Simula um processamento pesado em background
        processado = replace(pedido, status="processado")
        self._repository.save(processado)
        logger.info("Pedido %s processado com sucesso.", pedido.id)
```

---

## 🛠️ 2. Camada de Infraestrutura (`worker/infra/`)

A camada de Infraestrutura abriga os adaptadores técnicos do banco SQLite, as configurações de ambiente e o loop de escuta resiliente do protocolo AMQP.

### 📝 Configurações de Ambiente (`worker/infra/settings.py`)
Mapeia as variáveis do RabbitMQ e do SQLite de forma idêntica à API.

Crie o arquivo [worker/infra/settings.py](file:///worker/infra/settings.py) com o código abaixo:

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

### 📝 Gerenciador do Banco SQLite (`worker/infra/database.py`)
Controlará o ciclo de vida das conexões no banco compartilhado localmente com a API.

Crie o arquivo [worker/infra/database.py](file:///worker/infra/database.py) com o código abaixo:

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

### 📝 Topologia de Mensageria (`worker/infra/topology.py`)
Garante de forma declarativa e resiliente a declaração idêntica da topologia AMQP na inicialização do worker.

Crie o arquivo [worker/infra/topology.py](file:///worker/infra/topology.py) com o seguinte código:

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

### 📝 Consumidor Pika Resiliente com Backoff Exponencial (`worker/infra/consumer.py`)
Implementa a lógica do consumidor Pika com retentativa exponencial em caso de queda de rede e callbacks explícitas para controle seguro de Acks e Nacks.

Crie o arquivo [worker/infra/consumer.py](file:///worker/infra/consumer.py) com o código abaixo:

```python
import json
import logging
import time

import pika
from pika.adapters.blocking_connection import BlockingChannel
from pika.spec import Basic

from worker.domain.handler import MessageHandler
from worker.domain.models import Pedido
from worker.infra.settings import RabbitMQSettings
from worker.infra.topology import setup_topology

QUEUE_NAME: str = "pedidos_queue"
PREFETCH_COUNT: int = 1

MAX_RETRIES: int = 5
BACKOFF_BASE_SECONDS: int = 2

logger = logging.getLogger(__name__)


def build_connection_params(settings: RabbitMQSettings) -> pika.ConnectionParameters:
    return pika.ConnectionParameters(
        host=settings.host,
        port=settings.port,
        credentials=pika.PlainCredentials(settings.user, settings.password),
        heartbeat=60,
        blocked_connection_timeout=300,
    )


def connect_with_retry(params: pika.ConnectionParameters) -> pika.BlockingConnection:
    for attempt in range(MAX_RETRIES):
        try:
            return pika.BlockingConnection(params)
        except pika.exceptions.AMQPConnectionError:
            wait: int = BACKOFF_BASE_SECONDS**attempt
            logger.warning(
                "RabbitMQ indisponível. Tentativa %d/%d. Aguardando %ds.",
                attempt + 1,
                MAX_RETRIES,
                wait,
            )
            time.sleep(wait)
    raise RuntimeError(
        f"Não foi possível conectar ao RabbitMQ após {MAX_RETRIES} tentativas."
    )


def process_message(
    handler: MessageHandler,
    ch: BlockingChannel,
    method: Basic.Deliver,
    body: bytes,
) -> None:
    try:
        dados = json.loads(body)
        pedido = Pedido(**dados)
        handler.handle(pedido)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except (json.JSONDecodeError, pika.exceptions.AMQPError) as exc:
        logger.error("Erro ao processar mensagem: %s", exc, exc_info=True)
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)


def start_consuming(handler: MessageHandler, settings: RabbitMQSettings) -> None:
    params = build_connection_params(settings)
    connection = connect_with_retry(params)
    channel = connection.channel()

    setup_topology(channel)
    channel.basic_qos(prefetch_count=PREFETCH_COUNT)

    channel.basic_consume(
        queue=QUEUE_NAME,
        on_message_callback=lambda ch, method, properties, body: process_message(
            handler, ch, method, body
        ),
    )

    logger.info("Worker inicializado. Aguardando mensagens...")
    channel.start_consuming()
```

---

## ⚡ 3. Arquivo de Entrada (`worker/main.py`)

No ponto de entrada do Worker, configuramos os logs globais estruturados e instanciamos as dependências concretas ligadas às regras abstratas de negócio.

Crie o arquivo [worker/main.py](file:///worker/main.py) com o seguinte código:

```python
import logging

from worker.domain.handler import PedidoHandler
from worker.domain.repository import PedidoRepository
from worker.infra.consumer import start_consuming
from worker.infra.settings import get_settings

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)

if __name__ == "__main__":
    try:
        settings = get_settings()
        repository = PedidoRepository(db_path=settings.db_path)
        start_consuming(handler=PedidoHandler(repository), settings=settings)
    except KeyboardInterrupt:
        logging.getLogger(__name__).info("Worker encerrado pelo usuário.")
```

---

## 🚀 4. Executando e validando o Fluxo de Ponta a Ponta

1. Abra um terminal na raiz do seu workspace e inicie a API produtora:
   ```bash
   uv run task api-dev
   ```
2. Abra um segundo terminal e inicie o Worker de forma local utilizando a inicialização de módulo Python do `uv`:
   ```bash
   uv run python -m worker.main
   ```
3. **Cenário de Sucesso**: Em uma nova janela de terminal, envie um pedido válido via requisição POST HTTP:
   ```bash
   curl -X POST "http://localhost:8000/pedidos/" \
        -H "Content-Type: application/json" \
        -d '{"id": "ped-ok", "descricao": "Espada de Treino Jedi", "valor": 150.0}'
   ```
   * Verifique o log do Worker: ele deve registrar o início do processamento pesado (2 segundos), o salvamento no SQLite com status `"processado"`, e o envio do ACK.
   * Verifique na API consultando o endpoint `GET /pedidos/` (`curl http://localhost:8000/pedidos/`). O pedido com status `"processado"` listado deve ser!
4. **Cenário de Falha (DLX)**: Envie uma mensagem com payload JSON malformatado ou incompleto diretamente via RabbitMQ Management, ou altere temporariamente os tipos de entrada para forçar uma exceção de desserialização no Worker.
   * Verifique o log do Worker: ele capturará a exceção, enviará o `basic_nack(requeue=False)` de forma explícita, e a mensagem direcionada para a Dead Letter Exchange (`dlx_exchange`) automaticamente será!
   * No painel do RabbitMQ, comprove que a fila `dlx_pedidos` possui **1 mensagem** aguardando auditoria.

---

## 🧙‍♂️ Instruções do Mestre:

Pronto o seu Worker Consumidor Pika local e processando mensagens estar deve, jovem Padawan! O código das suas camadas de Domínio e Infraestrutura e a persistência compartilhada rodando apresentar você precisa!

O log de processamento de 2 segundos com o ACK de sucesso e a mensagem rejeitada na fila do `dlx_pedidos` no painel administrativo você comprovar deve para mim no chat!

> [!IMPORTANT]
> **Fluxo de Aprovação e Aprendizado**:
> Primeiro, a árvore física dos arquivos criados e a saída de sucesso/falha do terminal do Worker eu irei inspecionar.
> Após a sua demonstração prática impecável, perguntas reflexivas sobre os perigos do ACK automático (`auto_ack=True`) e de loops infinitos causados por NACK com re-enfileiramento (`requeue=True`) sem tratamento de falhas eu farei.
> 
> A sua compreensão lógica demonstrar você deve! Apenas após as respostas corretas estarem fornecidas, o sinal verde para o **Passo 5/6: Testes Automatizados** aceso será! Que a Força o acompanhe na escrita desse código!
