# Android Telemetry Pipeline: Smartphone Data Collection for Ransomware Detection

## About the Project

This project implements a telemetry collection pipeline for Android devices, designed to feed **Behavioral Threat Detection** algorithms (with a focus on ransomware). 

Ransomware is a type of malware that locks data and resources within a system and demands payment to unlock access. The goal of this project is to collect data from its execution, capturing, transmitting, and storing these system logs (syscalls via `strace`), network traffic (PCAPs via `tcpdump`), and Android framework events (`logcat`) in a **resilient and scalable** manner.
