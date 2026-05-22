# 🧪 Passo 5/6: Testes Automatizados, Mocks e Validação de Qualidade

Saudações, Padawan! A API e o seu Worker em rede local conversando e persistindo dados já estão! Orgulhoso do seu progresso na força eu estou, sim! No entanto, na Engenharia de Software profissional de elite, código sem **Testes Automatizados** considerado pronto nunca é!

Graças à nossa arquitetura em camadas do **DDD**, testar a lógica do domínio extremamente simples e rápido será, pois ela acoplamento físico com o RabbitMQ ou bancos concretos não possui! E para testarmos a infraestrutura e as rotas da nossa API sem dependermos do Broker ligado em rede, fixtures de **Mocking** profissional nós criaremos!

---

## 🏗️ 1. Estruturando a Pasta de Testes

No ecossistema unificado sincronizado com o `uv`, criaremos a nossa suite estruturando a pasta de testes de forma idêntica à referência:

```
rabbitmq-stack/
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Fixtures globais mockadas compartilhadas
│   ├── api/
│   │   ├── __init__.py
│   │   ├── domain/
│   │   │   ├── __init__.py
│   │   │   └── test_models.py   # Testes unitários do Pydantic da API
│   │   └── test_main.py         # Testes de integração mockada das rotas HTTP
│   └── worker/
│       ├── __init__.py
│       └── domain/
│           ├── __init__.py
│           ├── test_models.py   # Testes unitários do Dataclass do Worker
│           └── test_handler.py  # Testes unitários do Handler com mocks
```

---

## 🛠️ 2. As Fixtures Globais e Mocks (`tests/conftest.py`)

O arquivo `conftest.py` é um arquivo especial do `pytest` utilizado para declarar fixtures (funções auxiliares e mocks) que estarão disponíveis para todos os testes da suite de forma implícita. Criaremos os mocks estruturais da biblioteca Pika e a limpeza do cache de configurações.

Crie o arquivo [tests/conftest.py](file:///tests/conftest.py) contendo a especificação abaixo:

```python
from collections.abc import Generator
from unittest.mock import MagicMock

import pika
import pytest
from pika.adapters.blocking_connection import BlockingChannel
from pytest_mock import MockerFixture

from api.infra.settings import get_settings as api_get_settings
from worker.infra.settings import get_settings as worker_get_settings


@pytest.fixture(autouse=True)
def clear_settings_cache() -> Generator[None, None, None]:
    api_get_settings.cache_clear()
    worker_get_settings.cache_clear()
    yield
    api_get_settings.cache_clear()
    worker_get_settings.cache_clear()


@pytest.fixture
def mock_channel(mocker: MockerFixture) -> MagicMock:
    return mocker.MagicMock(spec=BlockingChannel)


@pytest.fixture
def mock_connection(mocker: MockerFixture, mock_channel: MagicMock) -> MagicMock:
    conn = mocker.MagicMock(spec=pika.BlockingConnection)
    conn.channel.return_value = mock_channel
    return conn
```

---

## 🧠 3. Testes Unitários de Domínio (`tests/api/` & `tests/worker/`)

O Domínio deve ser testado isoladamente de forma rápida. Validaremos as regras de negócio declaradas no Pydantic da API, as do `@dataclass(frozen=True)` do Worker e a lógica do caso de uso (`PedidoHandler`).

### 📝 Teste do Modelo Pydantic (`tests/api/domain/test_models.py`)
Crie o arquivo [tests/api/domain/test_models.py](file:///tests/api/domain/test_models.py) para certificar que o valor negativo ou nulo barreado de forma correta pelo validador do Pydantic será:

```python
import pytest
from pydantic import ValidationError

from api.domain.models import Pedido


def test_criar_pedido_valido():
    pedido = Pedido(id="1", descricao="Sabre de Luz", valor=1500.0)
    assert pedido.id == "1"
    assert pedido.valor == 1500.0
    assert pedido.status == "PENDENTE"


def test_pedido_com_valor_invalido_deve_lancar_erro():
    with pytest.raises(ValidationError) as exc_info:
        Pedido(id="2", descricao="Item inválido", valor=-10.0)

    assert "valor deve ser positivo" in str(exc_info.value)
```

### 📝 Teste do Dataclass do Worker (`tests/worker/domain/test_models.py`)
Crie o arquivo [tests/worker/domain/test_models.py](file:///tests/worker/domain/test_models.py) validando o comportamento da dataclass imutável:

```python
from worker.domain.models import Pedido


def test_criar_pedido_dataclass():
    pedido = Pedido(
        id="1", descricao="Notebook ASUS", valor=3500.0, status="processado"
    )
    assert pedido.id == "1"
    assert pedido.status == "processado"
```

### 📝 Teste do Caso de Uso / Handler (`tests/worker/domain/test_handler.py`)
Como o nosso `PedidoHandler` recebe uma dependência do repositório, podemos passar um mock cirúrgico dele sem tocar em bancos SQLite físicos durante os testes unitários.

Crie o arquivo [tests/worker/domain/test_handler.py](file:///tests/worker/domain/test_handler.py) com o código abaixo:

```python
from unittest.mock import Mock

from worker.domain.handler import PedidoHandler
from worker.domain.models import Pedido
from worker.domain.repository import PedidoRepository


def test_handler_processa_pedido_com_sucesso():
    # 1. Criar mock do repositório
    mock_repo = Mock(spec=PedidoRepository)
    handler = PedidoHandler(mock_repo)

    pedido = Pedido(id="1", descricao="Notebook ASUS", valor=3500.0, status="pendente")

    # 2. Executa o Handler
    handler.handle(pedido)

    # 3. Assegura que o repositório concreto de salvamento foi invocado
    mock_repo.save.assert_called_once()
    pedido_salvo = mock_repo.save.call_args[0][0]
    assert pedido_salvo.id == "1"
    assert pedido_salvo.status == "processado"
```

---

## 🛠️ 4. Testes de Integração da API HTTP (`tests/api/test_main.py`)

Para testar as rotas da API FastAPI de ponta a ponta sem disparar conexões físicas com o RabbitMQ, utilizaremos o `TestClient` injetando os mocks de Pika declarados no `conftest.py` para substituir a conexão do ciclo de vida global.

Crie o arquivo [tests/api/test_main.py](file:///tests/api/test_main.py) com o código abaixo:

```python
from unittest.mock import MagicMock

from fastapi.testclient import TestClient
from pytest_mock import MockerFixture

from api.infra.settings import RabbitMQSettings
from api.main import app

client = TestClient(app)


def test_health_check_endpoint():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}


def test_criar_pedido_com_sucesso(
    mocker: MockerFixture, mock_connection: MagicMock
) -> None:
    # 1. Mockar a conexão física do Pika no Lifespan da API
    mocker.patch(
        "api.main.pika.BlockingConnection", return_value=mock_connection
    )

    # Iniciar o ciclo de vida do lifespan para injetar os mocks no app.state
    with TestClient(app) as test_client:
        payload = {"id": "1", "descricao": "Notebook ASUS", "valor": 3500.0}

        # 2. Executa a requisição HTTP POST
        response = test_client.post("/pedidos/", json=payload)

        # 3. Asserções
        assert response.status_code == 202
        assert response.json() == {
            "mensagem": "Pedido enfileirado",
            "pedido_id": "1",
        }


def test_criar_pedido_com_erro_de_conexao(
    mocker: MockerFixture, mock_connection: MagicMock
) -> None:
    mocker.patch(
        "api.main.pika.BlockingConnection", return_value=mock_connection
    )
    
    # Simula erro de Runtime durante o envio da mensagem
    mocker.patch(
        "api.main.publish_pedido",
        side_effect=RuntimeError("Erro de comunicação com o Broker"),
    )

    with TestClient(app) as test_client:
        payload = {"id": "1", "descricao": "Notebook ASUS", "valor": 3500.0}
        response = test_client.post("/pedidos/", json=payload)
        
        assert response.status_code == 503
        assert response.json()["detail"] == "Erro de comunicação com o Broker"
```

---

## 🚀 5. Executando os Testes e Verificando Cobertura

1. Suite de testes configurada e sincronizada você tem.
2. Execute todos os testes a partir do terminal na raiz do workspace executando a tarefa rápida do `taskipy`:
   ```bash
   uv run task test
   ```
3. Para validar as métricas de cobertura de código (Code Coverage) assegurando que nenhuma ramificação lógica passou despercebida, execute a tarefa:
   ```bash
   uv run task test-cov
   ```
4. Verifique que todos os testes reportaram o status verde **PASSED** com cobertura mínima recomendada de 95%!

---

## 🧙‍♂️ Instruções do Mestre:

Toda a sua suite de testes verde e blindada estar deve, jovem Padawan! Mocks configurados de forma cirúrgica e o relatório de cobertura gerado apresentar você precisa, sim!

A saída do terminal da sua tarefa `test-cov` demonstrando a integridade da API e do Worker você me mostrar no chat deve!

> [!IMPORTANT]
> **Fluxo de Aprovação e Aprendizado**:
> Primeiro, a execução limpa dos seus testes automatizados no terminal eu irei validar.
> Após a comprovação verde prática, perguntas reflexivas sobre os papéis das fixtures do pytest e a importância de isolar os testes do banco de dados físico SQLite eu farei.
> 
> Responder com clareza você deve! Apenas após a sua demonstração teórica bem-sucedida, a autorização final para o **Passo 6/6: Simulação de Produção e Dockerização Integrada** concedida será! Que a cobertura verde com você esteja!
