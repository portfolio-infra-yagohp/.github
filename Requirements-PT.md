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
