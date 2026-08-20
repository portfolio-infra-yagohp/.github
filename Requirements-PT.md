# Requisitos do Sistema - Pipeline de Telemetria Android

## 1. Requisitos Funcionais (RF)

### 1.1. Camada de Coleta (Dispositivo Android)

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RF01** | Iniciar Sessão de Coleta | O conjunto de bash scripts podem iniciar uma sessão de coleta para um aplicativo específico (`app_name`) via script `monitor_start.sh`, informando o ID do cartão SD e parâmetros de tamanho máximo de log para rotação. |
| **RF02** | Coletar Syscalls (Strace) | O script deve anexar o resultado de `strace` ao PID (Process ID) do aplicativo alvo para capturar chamadas de sistema relacionadas a arquivos, processos, rede e execução (`execve`). |
| **RF03** | Coletar Tráfego de Rede (PCAP) | O script deve capturar todo o tráfego de rede da interface utilizando `tcpdump`, com rotação automática de arquivos a cada 60 segundos para evitar arquivos excessivamente grandes. |
| **RF04** | Coletar Logs do Framework (Logcat) | O script deve capturar logs do Android (`logcat`) filtrando eventos relevantes (ActivityManager, ServiceManager, PackageManager, PowerManager, Binder, etc.). |
| **RF05** | Persistência Local (Resiliência) | O sistema deve armazenar os logs de texto em uma fila local (`pending_queue.jsonl`) e os PCAPs em uma pasta local (`pending_pcaps/`) antes do envio, garantindo que dados não sejam perdidos em caso de queda de rede. |
| **RF06** | Compressão e Rotação de Logs | O sistema deve compactar (`.tar.gz`) e rotacionar diretórios de log antigos para liberar espaço em disco, conforme configurado. |
| **RF07** | Verificação de Espaço em Disco | O sistema deve monitorar o espaço disponível no SD Card e interromper a coleta se o espaço livre cair abaixo do limite mínimo configurado (ex: 1GB). |

### 1.2. Camada de Ingestão (API .NET + Load Balancer)

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RF08** | Receber Logs em Lote | A API deve expor o endpoint `POST /api/v1/ingest/logs` para receber múltiplos eventos de telemetria em uma única requisição. |
| **RF09** | Validar Autenticação | A API deve validar a `X-API-Key` enviada no header de cada requisição, rejeitando requisições não autenticadas. |
| **RF10** | Registrar Dispositivos | A API deve registrar automaticamente novos dispositivos no DynamoDB (tabela `Devices`) na primeira vez que enviarem dados. |
| **RF11** | Mander mecanismo de cache | A API deve gerenciar dados em cache para evitar muitas consultas diretas ao DynamoDB. |
| **RF12** | Validar Metadados | A API deve validar a presença dos campos obrigatórios (`device_id`, `log_type`, `raw`) na mensagem. |
| **RF13** | Publicar na Fila de Processamento | A API deve publicar cada lote validado em uma fila, para processamento assíncrono, retornando imediatamente `202 Accepted` ao dispositivo. |

### 1.3. Camada de Processamento

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RF13** | Processar Logs de Texto (Lambda-Logs) | A Lambda-Logs, acionada pela SQS, deve consumir as mensagens, processar o payload e indexar cada evento no Elasticsearch. |
| **RF14** | Extrair Metadados de PCAP (Lambda-PCAP) | A Lambda-PCAP, acionada por evento S3, deve baixar o arquivo `.pcap`, extrair metadados de rede (5-tuplas, DNS, TLS-SNI) utilizando `tshark`, e indexá-los no Elasticsearch. |
| **RF15** | Deduplicação de Eventos | A Lambda deve verificar o Redis (hash do evento) antes de indexar para evitar duplicatas em caso de reprocessamento de mensagens da SQS ou S3. |
| **RF16** | Enriquecimento de Dados | A Lambda deve enriquecer os eventos com metadados do dispositivo (consultando o Redis ou DynamoDB) antes de indexar. |

### 1.4. Camada de Armazenamento e Estado

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RF17** | Indexar Dados no Elasticsearch | O sistema deve indexar logs de texto e metadados de rede no Elasticsearch em Data Streams separados (`logs-strace`, `logs-logcat`, `logs-network`). |
| **RF18** | Armazenar PCAPs no S3 | O sistema deve armazenar os arquivos PCAP brutos no Amazon S3, permitindo download sob demanda para análises forenses profundas. |
| **RF19** | Gerenciar Sessões no DynamoDB | O sistema deve manter o estado das sessões ativas no DynamoDB (tabela `Sessions`), registrando início, heartbeats e término das coletas. |
| **RF20** | Gerenciar Regras no DynamoDB | O sistema deve permitir o gerenciamento de regras de coleta por aplicativo (tabela `AppRules`), definindo quais syscalls ou eventos devem ser filtrados. |


### 1.5. Camada de DevOps

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RF21** | Consultar Dados (Kibana) | O sistema deve disponibilizar dashboards no Kibana para que pesquisadores consultem eventos históricos por `device_id`, `session_id` ou buscas full-text no campo `raw`. |
| **RF22** | Monitorar Métricas (Grafana) | O sistema deve expor métricas de performance (tamanho da fila SQS, latência da API, uso de CPU das EC2s) em dashboards do Grafana. |
| **RF23** | Executar CI/CD (Jenkins/Rundeck) | O sistema deve possuir pipelines automatizadas no Jenkins para build, testes (Smoke Tests) e deploy da API .NET e das Lambdas. |
| **RF24** | Executar Backups e Deployments (Rundeck) | O sistema deve executar snapshots agendados do Elasticsearch e rotinas de limpeza do S3 via Rundeck. |


---

