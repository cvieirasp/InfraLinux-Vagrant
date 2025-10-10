# Laboratório de Linux (Vagrant)

Este repositório contém um laboratório prático originado à partir das práticas do curso de Fundamentos de Linux, com foco em levantar um ambiente mínimo composto por três máquinas virtuais usando Vagrant:

- **dns:** Rocky Linux 9 com BIND (named) instalado e serviço ativo.
- **web:** Rocky Linux 9 com Apache (httpd) instalado e serviço ativo.
- **client:** Ubuntu 22.04 com utilitários de rede.

## Objetivos do laboratório

- Praticar fundamentos de Linux (serviços, systemd, rede, permissões, logs).
- Entender o papel de cada servidor no ecossistema:
  - Servidor DNS autoritativo (BIND) - conceito de zonas direta/reversa.
  - Servidor Web (Apache) - conceitos de serviço e diretórios padrão.
  - Cliente Ubuntu - Máquina cliente para testes dos serviços dentro da rede.
- Operar VMs com Vagrant: subir, acessar, pausar e destruir o ambiente de forma reprodutível.

## Topologia e parâmetros

| Hostname              | SO            | Função        | IP (rede privada) |
| --------------------- | ------------- | ------------- | ----------------- |
| dns.masterlab.com     | Rocky Linux 9 | DNS (BIND)    | 10.10.10.10       |
| web.masterlab.com     | Rocky Linux 9 | WEB (Apache)  | 10.10.10.20       |
| client.masterlab.com  | ubuntu 22.04  | DNS (BIND)    | 10.10.10.30       |

> As VMs usam rede privada (10.10.10.0/24) apenas para comunicação entre elas e o adaptador NAT padrão do Vagrant é mantido para acesso à internet/updates.

## Pré-requisitos

- VirtualBox (ou outro provider compatível com Vagrant)
- Vagrant 2.x

## Como Executar

1. Obtenha o projeto realizando download/clonagem do repositório para uma pasta local.
2. Suba o ambiente no diretório onde encontra-se o arquivo ```Vagrantfile``` utilizando os seguintes comandos:
  - ```vagrant up``` (sobe todas as VMs) ou
  - ```vagrant dns```, ```vagrant up web```, ```vagrant up client``` (uma por vez).
3. Acesse as VMs:
  -  ```vagrant ssh dns```
  -  ```vagrant ssh web```
  -  ```vagrant ssh client```
4. Encerrar/suspender quando terminar
  - ```vagrant halt``` (desliga) ou ```vagrant suspend``` (suspende).
  - ```vagrant destroy``` (remove as VMs) quando quiser recriar do zero.

## Configurações dos Servidores

- [Configuração DNS Server](docs/DNSSERVER.md)
- [Configuração Web Server](docs/WEBSERVER.md)