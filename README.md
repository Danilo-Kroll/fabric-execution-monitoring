# Fabric Execution Monitoring

Monitoramento e histórico de execuções do **Microsoft Fabric e Power BI** utilizando um Notebook do Microsoft Fabric, APIs REST e uma tabela Delta em Lakehouse.

O objetivo do projeto é criar uma visão histórica das execuções do ambiente, permitindo analisar:

* execuções concluídas;
* falhas;
* cancelamentos;
* execuções desabilitadas;
* duração;
* horário das execuções;
* tipo de acionamento;
* erros retornados pelas APIs;
* comportamento por workspace;
* comportamento por artefato;
* comportamento por tipo de item.

A solução persiste os dados em um Lakehouse, evitando depender apenas do histórico recente disponibilizado pelas APIs.

---

## Visão geral

O Microsoft Fabric possui informações de monitoramento distribuídas entre diferentes APIs.

Este projeto centraliza esses dados em uma única tabela histórica.

```text
Microsoft Fabric
       │
       ├── Fabric REST API
       │       │
       │       ├── Pipelines
       │       ├── Dataflows Gen2
       │       ├── Notebooks
       │       ├── Copy Jobs
       │       └── Outros Fabric Jobs
       │
       ├── Power BI REST API
       │       │
       │       ├── Semantic Models
       │       └── Dataflows Gen1
       │
       ▼
Fabric Notebook
       │
       ├── coleta
       ├── normalização
       ├── tratamento de erros
       └── MERGE
       │
       ▼
Lakehouse
       │
       ▼
fabric_monitoramento_historico
       │
       ▼
Power BI
```

---

## Por que criar um histórico próprio?

As APIs utilizadas retornam um conjunto recente de execuções.

Isso é suficiente para acompanhamento operacional, mas não necessariamente para análises históricas de longo prazo.

O notebook deste projeto coleta periodicamente essas informações e grava os registros em uma tabela Delta.

Dessa maneira é possível manter meses ou anos de histórico independentemente da retenção disponibilizada pelas APIs.

---

## O que é monitorado?

### Microsoft Fabric Jobs

Através da Fabric REST API são coletadas execuções de artefatos como:

* Pipeline;
* Dataflow Gen2;
* Notebook;
* Copy Job;
* Spark Job;
* outros itens que disponibilizem histórico através da Job Scheduler API.

Entre as informações disponíveis estão:

```text
job_type
invoke_type
status
start_time_utc
end_time_utc
failureReason
rootActivityId
```

---

## Modelos Semânticos

Os modelos semânticos são consultados através da Power BI REST API.

Além do status da atualização, o histórico pode retornar detalhes importantes de falha através do campo:

```text
serviceExceptionJson
```

O notebook tenta transformar esse conteúdo em três campos:

```text
error_code
error_message
error_raw
```

Exemplo:

```text
error_code
DM_GWPipeline_Gateway_MashupDataAccessError | SQL 10054

error_message
A transport-level error has occurred when receiving results
from the server.

error_raw
JSON completo retornado pela API.
```

Isso facilita muito a investigação de problemas de:

```text
Gateway
Rede / TCP
SQL Server
Credenciais
Capacidade
Modelo
Refresh
```

---

## Dataflow Gen1

Dataflows Gen1 são coletados separadamente através da Power BI REST API.

São armazenadas informações como:

```text
refreshType
startTime
endTime
status
```

Nem todas as APIs disponibilizam o mesmo nível de detalhamento de erro.

Por esse motivo, algumas execuções podem possuir status de falha sem uma mensagem detalhada em `error_message`.

---

## Estrutura da tabela

A tabela principal sugerida é:

```text
fabric_monitoramento_historico
```

Principais campos:

| Campo                | Descrição                                   |
| -------------------- | ------------------------------------------- |
| `data_coleta_utc`    | Momento em que o notebook realizou a coleta |
| `workspace_id`       | ID do workspace                             |
| `workspace_name`     | Nome do workspace                           |
| `item_id`            | ID do artefato                              |
| `item_name`          | Nome do artefato                            |
| `item_type`          | Tipo amigável do artefato                   |
| `item_type_original` | Tipo original retornado pela API            |
| `origem_api`         | API responsável pelo registro               |
| `execution_id`       | Identificador da execução                   |
| `job_type`           | Tipo de job/refresh                         |
| `invoke_type`        | Forma como a execução foi iniciada          |
| `status`             | Status da execução                          |
| `start_time_utc`     | Início da execução                          |
| `end_time_utc`       | Fim da execução                             |
| `duration_seconds`   | Duração em segundos                         |
| `error_code`         | Código resumido do erro                     |
| `error_message`      | Mensagem tratada                            |
| `error_raw`          | Retorno completo da API                     |
| `activity_id`        | Activity ID quando disponível               |
| `request_id`         | Request ID quando disponível                |
| `chave_execucao`     | Chave utilizada para identificar a execução |

---

## Horários

O projeto mantém os horários originalmente em **UTC**.

Exemplo:

```text
start_time_utc
2026-08-21 10:00:00
```

A conversão para o horário local deve ser realizada preferencialmente na camada de consumo.

Exemplo para Brasília:

```text
UTC       10:00
Brasília  07:00
```

Essa abordagem evita diferenças de interpretação entre:

```text
Fabric API
Spark
Delta Lake
SQL Analytics Endpoint
Power BI
```

e mantém UTC como referência técnica única.

---

## Controle de duplicidade

O notebook não realiza apenas `append`.

Cada execução recebe uma chave:

```text
chave_execucao
```

Exemplo conceitual:

```text
ORIGEM | WORKSPACE | ITEM | EXECUTION_ID
```

Na gravação é realizado um `MERGE`.

Assim, se uma execução estiver inicialmente:

```text
InProgress
```

e na próxima coleta aparecer:

```text
Completed
```

o registro existente pode ser atualizado em vez de uma segunda linha ser criada.

Isso também permite executar o notebook várias vezes ao dia sem duplicar todo o histórico.

---

## Autenticação

Na implementação simples deste projeto, o Notebook Fabric utiliza:

```python
notebookutils.credentials.getToken("pbi")
```

O token é utilizado nas chamadas às APIs do Microsoft Fabric e Power BI.

O acesso aos workspaces e artefatos continuará respeitando as permissões da identidade utilizada para executar o notebook.

Para ambientes corporativos que exigem autenticação mais controlada ou execução utilizando identidade de aplicação, a autenticação pode ser evoluída posteriormente para Microsoft Entra ID / MSAL.

---

## Execução automática

Depois de validado manualmente, o notebook pode ser executado através de agendamento.

Exemplo:

```text
05:00
17:00
```

Não existe uma frequência obrigatória.

A frequência ideal depende da quantidade de execuções existente no ambiente.

Ambientes com muitos jobs ao longo do dia podem utilizar intervalos menores.

Como os registros são tratados através de `MERGE`, aumentar a frequência da coleta não significa necessariamente gerar duplicidades.

---

## Status

As APIs podem retornar diferentes valores de status.

Para facilitar a visualização no Power BI, eles podem ser agrupados em categorias como:

```text
Sucesso
Falha
Em execução
Cancelado
Desabilitado
Indefinido
```

Exemplos de valores encontrados:

```text
Completed
Success
Failed
InProgress
NotStarted
Cancelled
Disabled
Unknown
```

Recomenda-se manter sempre o status original na tabela histórica.

---

## Tipo de execução

O campo:

```text
invoke_type
```

ajuda a identificar como uma execução foi iniciada.

Alguns exemplos:

```text
Scheduled
Manual
OnDemand
ViaApi
ViaXmlaEndpoint
DataFactory
DirectLakeFraming
OneLakeIntegration
```

Na camada de relatório esses valores podem ser agrupados em:

```text
Agendada
Manual
Automação / API
Interna Fabric
```

---

## Dashboard

Com a tabela histórica é possível criar um painel operacional de monitoramento.

Exemplo de indicadores:

```text
Total de Execuções
Taxa de Sucesso
Taxa de Falha
Execuções Canceladas
Execuções Desabilitadas
```

Também podem ser analisados:

```text
Execuções por horário
Falhas por horário
Erros mais frequentes
Falhas por artefato
Falhas por workspace
Status por dia
Execuções por tipo de artefato
Execuções agendadas x manuais x API
```

Uma imagem de exemplo pode ser adicionada em:

```text
docs/dashboard.png
```

---

## Possível classificação de erros

Para simplificar o acompanhamento pela equipe de infraestrutura, os códigos podem posteriormente ser agrupados em categorias.

Exemplo:

```text
Rede / TCP
Gateway
Credencial
Capacidade
Semantic Model
Data Factory
Cancelamento
Outro
```

O código técnico original deve continuar armazenado em:

```text
error_code
```

e o retorno integral em:

```text
error_raw
```

---

## Pré-requisitos

É necessário possuir:

* Microsoft Fabric;
* um Workspace;
* um Lakehouse;
* um Notebook Fabric;
* acesso aos workspaces que serão monitorados;
* permissões suficientes para consultar os artefatos através das APIs;
* Power BI, caso queira construir a camada de visualização.

---

## Estrutura do projeto

```text
fabric-execution-monitoring/
│
├── README.md
├── LICENSE
│
├── notebooks/
│   └── nb_fabric_execution_monitoring.ipynb
│
├── docs/
│   ├── dashboard.png
│   └── architecture.png
│
└── .gitignore
```

---

## Instalação

### 1. Crie um Lakehouse

Exemplo:

```text
LH_FabricMonitoring
```

### 2. Importe o notebook

Importe:

```text
notebooks/nb_fabric_execution_monitoring.ipynb
```

para um Workspace do Microsoft Fabric.

### 3. Anexe o Lakehouse

Adicione o Lakehouse criado como Lakehouse padrão do notebook.

### 4. Execute o notebook

Na primeira execução será criada a tabela:

```text
fabric_monitoramento_historico
```

### 5. Valide os dados

Confira principalmente:

```text
workspace_name
item_name
item_type
invoke_type
status
start_time_utc
end_time_utc
error_code
error_message
```

### 6. Agende a coleta

Depois da validação, configure a execução automática na frequência adequada ao seu ambiente.

---

## Segurança

Antes de publicar exemplos, logs ou screenshots em um repositório público, remova informações internas como:

```text
IP de servidores
nomes de bancos
IDs internos
nomes sensíveis de workspaces
Activity IDs reais
Request IDs reais
mensagens contendo endereços internos
tokens
segredos
credenciais
```

Nunca armazene tokens ou secrets diretamente no código publicado.

---

## Limitações

Este projeto não pretende substituir todas as ferramentas de observabilidade do Microsoft Fabric.

O objetivo é fornecer uma solução simples para criar um **histórico operacional centralizado de execuções**.

Algumas limitações:

* diferentes tipos de artefato podem retornar informações diferentes;
* nem toda API disponibiliza mensagem detalhada de erro;
* a visibilidade depende das permissões da identidade que executa a coleta;
* novas funcionalidades e tipos de item do Fabric podem exigir atualização da lista de artefatos suportados;
* métricas de capacidade não fazem parte do escopo inicial.

---

## Roadmap

Possíveis evoluções:

```text
[ ] Classificação automática de erros
[ ] Monitoramento de capacidade
[ ] Histórico de agendamentos configurados
[ ] Alertas para falhas críticas
[ ] Detecção de falhas simultâneas
[ ] Identificação de artefatos sem atualização
[ ] Dashboard Power BI de exemplo
[ ] Suporte a novos tipos de artefato
[ ] Service Principal / MSAL
```

---

## Contribuições

Contribuições, sugestões e melhorias são bem-vindas.

Se encontrar um comportamento diferente em determinado tipo de artefato do Fabric, abra uma issue descrevendo:

```text
Tipo do artefato
API utilizada
Status retornado
Comportamento esperado
Comportamento observado
```

---

## Licença

Sugestão: **MIT License**.

Isso permite que outras pessoas utilizem, modifiquem e evoluam o projeto, mantendo o aviso de copyright e licença.

---

## Disclaimer

Este é um projeto comunitário e independente.

Não é um produto oficial da Microsoft e não possui vínculo com a Microsoft.
