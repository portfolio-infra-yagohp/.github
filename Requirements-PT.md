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
