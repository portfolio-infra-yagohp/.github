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

## 2. Requisitos Não Funcionais (RNF)

### 2.1. Performance

| ID | Requisito | Descrição | Métrica |
| :--- | :--- | :--- | :--- |
| **RNF01** | Baixa Latência da API | A API .NET deve responder em menos de 200ms, pois apenas publica na SQS e retorna (não realiza processamento pesado). | P95 < 200ms |
| **RNF02** | Throughput da Ingestão | O sistema deve suportar muitas requisições para a ingestão por segundo (ainda a definir), distribuídos entre as instâncias da API. | > 10k eventos/s ou 200 req/s |
| **RNF03** | Indexação Rápida | A Lambda-Logs deve processar e indexar lotes de eventos no Elasticsearch em menos de 5 segundos para dados em "tempo real" (hot). | P95 < 5s |
| **RNF04** | Cache de Baixa Latência | O Redis deve responder às consultas de cache (validação de dispositivo) em menos de 1ms para não se tornar um gargalo. | P99 < 1ms |

### 2.2. Resiliência e Tolerância a Falhas

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RNF05** | Zero Data Loss | O sistema deve garantir que nenhuma linha de log ou PCAP seja perdida, mesmo durante falhas de rede ou indisponibilidade da AWS. (Garantido por fila local + SQS + DLQ). |
| **RNF06** | Graceful Degradation (Cache) | Se o Redis (EC2) falhar, a API deve consultar diretamente o DynamoDB (via Circuit Breaker + Fallback) e continuar operacional. |
| **RNF07** | Recuperação de Falhas (DLQ) | Mensagens que falharem 3x no processamento da Lambda devem ser enviadas para a Dead Letter Queue (DLQ) para inspeção manual e re-drive. |
| **RNF08** | Alta Disponibilidade da API | A API .NET deve rodar em pelo menos 2 instâncias EC2 com Load Balancer NGINX para tolerar falha de uma instância. |


### 2.3. Segurança

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RNF09** | Autenticação por API Key | Todas as requisições para a API de ingestão devem ser autenticadas via `X-API-Key`. |
| **RNF10** | Criptografia em Trânsito | Todas as comunicações entre o dispositivo Android e a AWS (API e S3) devem utilizar TLS/HTTPS. |
| **RNF11** | Controle de Acesso (IAM) | Os serviços AWS (Lambda, EC2) devem executar com papéis IAM com privilégios mínimos necessários (Least Privilege). |
| **RNF12** | Rate Limiting | A API deve limitar o número de requisições por `device_id` (ex: 100 req/min) utilizando Redis para prevenir abusos. |


### 2.4. Escalabilidade

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RNF13** | Escala Horizontal da API | O número de instâncias EC2 da API .NET deve poder ser aumentado horizontalmente para lidar com picos de tráfego. |
| **RNF14** | Escala Automática das Lambdas | A Lambda-Text deve escalar automaticamente com o aumento de mensagens na fila SQS (gerenciado pela AWS). |
| **RNF15** | Elasticidade do Elasticsearch | O cluster Elasticsearch deve ser dimensionado conforme o volume de dados (ex: via ILM e ajuste de shards). |

### 2.5. Manutenibilidade e Qualidade de Código

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RNF16** | Versionamento Semântico | Todos os artefatos (API, Lambdas, Scripts) devem ser versionados utilizando SemVer (v1.2.3). |
| **RNF17** | Rastreabilidade (Tags Git) | Cada release deve ser marcada com uma tag no Git, garantindo que o código fonte de uma versão seja rastreável até o artefato implantado no S3. |
| **RNF18** | Cobertura de Testes | O código da API .NET e das Lambdas deve ter cobertura mínima de 85% em testes unitários e incluir Smoke Tests no pipeline CI/CD. |
| **RNF19** | Infraestrutura como Código (IaC) | Toda a infraestrutura AWS (EC2, SQS, S3, Lambda, etc.) deve ser provisionada via Terraform ou AWS CDK para garantir reprodutibilidade. |


### 2.6. Custo e Retenção de Dados

| ID | Requisito | Descrição |
| :--- | :--- | :--- |
| **RNF20** | Otimização de Custos | Utilizar serverless (Lambda, SQS) para processamento variável e armazenar dados brutos (PCAPs) no S3, mais barato que Elasticsearch. |
| **RNF21** | Política de Retenção (ILM) | Dados no Elasticsearch devem ser movidos para "Warm" e "Cold" conforme a idade (ex: 7 dias Warm, 30 dias Cold) e deletados após 90 dias. |
| **RNF22** | Política de Ciclo de Vida do S3 | PCAPs no S3 devem ser movidos para camadas Standard-IA (30 dias) e Glacier Deep Archive (180 dias) para reduzir custos. |
| **RNF23** | Retenção de Artefatos | Manter as últimas 5 versões de artefatos no S3; versões mais antigas devem ser movidas para camadas frias ou deletadas. |




