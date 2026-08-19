# Android Telemetry Pipeline: Coleta de Dados de Smartphone para Detecção de Ransomware

## Sobre o Projeto

Este projeto implementa um pipeline de coleta de telemetria de dispositivos Android, projetado para alimentar algoritmos de **Detecção Comportamental de Ameaças** (foco em ransomware). 

Um ransomware é um tipo de malware que bloqueia os dados e recursos em um sistema e exige pagamento para desbloquear o acesso. O objetivo deste projeto é coletar dados de sua  execução, caputrando, transmitindo e armazenando esses logs de sistema (syscalls via `strace`), tráfego de rede (PCAPs via `tcpdump`) e eventos do framework Android (`logcat`) de forma **resiliente e escalável**.
