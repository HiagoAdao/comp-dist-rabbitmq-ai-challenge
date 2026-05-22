# ⚔️ RabbitMQ Stack — Imersão em Arquiteturas Event-Driven com IA

Seja bem-vindo ao ambiente de criação da **Stack de Mensageria Resiliente**, utilizando uma **IA de Mentoria** para guiar o aprendizado em Python, FastAPI, Pika (RabbitMQ) e Docker Compose.

> 💡 **Nota:** A Inteligência Artificial neste projeto é utilizada **exclusivamente para o ensino** e auxílio na criação do código. A stack de mensageria assíncrona a ser construída é de produção real e orquestrada de forma autoritária, desacoplada e resiliente.

Este repositório foi montado como uma **Jornada de Aprendizado Pedagógica (#anti-vibe-coding)**. O objetivo não é apenas gerar código de forma caótica, mas permitir que você (o Padawan) aprenda a construir um sistema distribuído robusto com a supervisão atenta do agente inteligente **Jedi da Mensageria**.

---

## 🏗️ Estrutura do ambiente

A verdadeira "mágica" arquitetural e pedagógica não está solta na raiz, mas sim encapsulada nestes arquivos e diretórios essenciais:

```text
comp-dist-rabbitmq-ai-challenge/
├── AGENTS.md                  # 🧙‍♂️ Orquestrador: Arquivo raiz que converte a IA no "Jedi da Mensageria".
├── .ai-instructions/          # 🧠 Cérebro do Jedi: Regras, arquitetura e restrições.
└── README.md                  # 📖 Este arquivo, contendo as instruções iniciais e de uso.
```

O diretório `.ai-instructions/` contém o gabarito arquitetural do produtor (API), do consumidor (Worker) e do ambiente (Docker). **Evite inspecionar** diretamente estes arquivos se desejar ter a experiência educacional completa de ser guiado e desafiado pela IA.

---

## 🤖 Dinâmica de ensino (AGENTS.md)

Este projeto utiliza o padrão [AGENTS.md](https://agents.md/) para definir e orquestrar a inteligência artificial. Essa abordagem garante que a IA não atue apenas como um gerador de código, mas como um tutor ativo que:

1. **Mantém a Persona:** Age rigorosamente como o Jedi da Mensageria.
2. **Segue um Framework Cognitivo:** Aplica uma metodologia pedagógica modularizada com **Indicador de Progresso** visual a cada interação.
3. **Perguntas Ativas:** Interrompe o aluno a cada final de passo com questionamentos reflexivos e profundos antes de liberar a próxima etapa.
4. **Erradica o "Vibe Coding":** Impõe planejamento e reflexão antes de qualquer linha de código ser gerada, garantindo que o Padawan entenda o "porquê" de cada implementação de AMQP e de design estratégico.
5. **Quiz de Fixação:** Ao concluir a jornada, aplica um Quiz interativo para comprovar a consolidação dos conceitos.

---

## 🚀 Primeiros passos: Ativando sua imersão

Para dar início ao desenvolvimento, você precisará clonar o repositório e utilizar uma IDE acoplada a um assistente de IA (ex: Cursor, Cline, Roo Code, etc.) configurada no diretório do projeto:

1. **Clone o repositório** em sua máquina local:
   ```bash
   git clone https://github.com/HiagoAdao/comp-dist-rabbitmq-ai-challenge.git
   ```
2. **Abra a pasta** `comp-dist-rabbitmq-ai-challenge` na sua IDE de escolha.
3. **Inicie o Agente de IA** (Chat / Composer).
4. **Primeira Interação:** Basta enviar um simples "Olá" no chat inicial! O agente fará a leitura autônoma da cadeia de diretivas e arquivos ocultos para se inicializar.

Deste ponto em diante, apenas siga o plano arquitetural do Jedi da Mensageria para codificar o projeto!

---

## 🏆 O que você terá construído ao final do treinamento?

Ao concluir esta jornada de mentoria, você terá em mãos um ambiente de mensageria assíncrona resiliente de nível corporativo, contendo as seguintes funcionalidades:

* 📡 **API FastAPI (Produtora)**: Recebe requisições HTTP de criação de pedidos, realiza validações com Pydantic, e as enfileira de forma confiável no RabbitMQ sob o protocolo *Publisher Confirms*.
* ⚙️ **Worker Consumidor Python**: Um microsserviço independente baseado no driver *Pika* que processa tarefas assíncronas em segundo plano com fair dispatch (`prefetch_count=1`).
* 🛡️ **Resiliência e Tolerância a Falhas (DLX)**: Tratamento de exceções robusto usando basic_nack manual com `requeue=False`, direcionando pedidos inválidos ou problemáticos automaticamente para uma Dead Letter Queue (`dlx_pedidos`) para posterior auditoria.

### 🚀 Principais habilidades adquiridas

Finalizar este treinamento comprovará seu domínio tático nas seguintes trincheiras:

* **DDD e Arquitetura Limpa**: Separação estrita de responsabilidades entre regras de Domínio (`domain/`) e detalhes de infraestrutura (`infra/`) aplicando Inversão de Dependências (DIP) e Responsabilidade Única (SRP).
* **Confiabilidade de Entrega**: Domínio prático de conceitos de AMQP (Publisher Confirms, Fair Dispatch, ACKs/NACKs e Dead Letter Exchange).
* **Orquestração de Microsserviços**: Criação de múltiplos serviços independentes e interconectados em rede local isolada usando **Docker e Docker Compose** com Healthchecks resilientes.
* **Testes de Integração e Mocks**: Testar regras de domínio de forma 100% isolada e infraestrutura de mensageria mockando canais e filas com `pytest`.
* **DevOps Local e REST Client**: Testes automatizados de carga de concorrência com `requests.http` na API de produção e coleta de métricas em tempo real na API de Gerenciamento do RabbitMQ.

> *"Em um lugar escuro nos encontramos, e um pouco mais de conhecimento ilumina nosso caminho." — Yoda*
