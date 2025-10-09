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

## Configuração do Servidor DNS

**1 - Acesse a VM e prepare o ambiente**

```shell
vagrant ssh dns
sudo -i
```

Faça um backup do config padrão (boa prática):

```shell
cp -a /etc/named.conf /etc/named.conf.bak.$(date +%F-%H%M%S)
```

Crie as pastas de logs já com as devidas permissões:

```shell
mkdir /var/log/named
chgrp named /var/log/named
chmod 775 /var/log/named
```

Edite o arquivo: ```named.conf```

```shell
vim /etc/named.conf
```

Cole o conteúdo abaixo:

```
acl "lab-net" { 127.0.0.1; 10.10.10.0/24; };

options {
  listen-on port 53 { 127.0.0.1; 10.10.10.10; };
  listen-on-v6 { none; };

  // ---- Servidor DNS Autoritativo puro ----
  recursion no; // desliga recursão
  allow-recursion { none; }; // ninguém faz recursão
  allow-query { 10.10.10.0/24 }; // quem pode consultar suas ZONAS
  allow-query-cache { none; }; // não responde cache

  // Nenhum forwarders configurado
  // forwarders { };

  // DNSSEC (não é necessário para autoritativo simples sem assinar zonas)
  dnssec-validation no;

  directory "/var/named";
  bindkeys-file "/etc/named.root.key";
  managed-keys-directory "/var/named/dynamic";
};

logging {
    channel default_file {
    file "/var/log/named/default.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel general_file {
    file "/var/log/named/general.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel database_file {
    file "/var/log/named/database.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel security_file {
    file "/var/log/named/security.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel config_file {
    file "/var/log/named/config.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel resolver_file {
    file "/var/log/named/resolver.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel xfer-in_file {
    file "/var/log/named/xfer-in.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel xfer-out_file {
    file "/var/log/named/xfer-out.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel notify_file {
    file "/var/log/named/notify.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel client_file {
    file "/var/log/named/client.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel unmatched_file {
    file "/var/log/named/unmatched.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel queries_file {
    file "/var/log/named/queries.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel network_file {
    file "/var/log/named/network.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel update_file {
    file "/var/log/named/update.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel dispatch_file {
    file "/var/log/named/dispatch.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel dnssec_file {
    file "/var/log/named/dnssec.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  channel lame-servers_file {
    file "/var/log/named/lame-servers.log" versions 3 size 5m;
    severity dynamic;
    print-time yes;
  };
  category default { default_file; };
  category general { general_file; };
  category database { database_file; };
  category security { security_file; };
  category config { config_file; };
  category resolver { resolver_file; };
  category xfer-in { xfer-in_file; };
  category xfer-out { xfer-out_file; };
  category notify { notify_file; };
  category client { client_file; };
  category unmatched { unmatched_file; };
  category queries { queries_file; };
  category network { network_file; };
  category update { update_file; };
  category dispatch { dispatch_file; };
  category dnssec { dnssec_file; };
  category lame-servers { lame-servers_file; };
};

zone "." IN {
  type hint;
  file "named.ca";
}

// ===== Zonas autoritativas =====
zone "masterlab.com" IN {
  type master;
  file "masterlab.com.zone";
};

zone "10.10.10.in-addr.arpa" IN {
  type master;
  file "masterlab.com.rev.zone";
};
```

Valide de a configuração está correta:

```shell
named-checkconf
```

Crie o arquivo de zona chamado ```masterlab.com.zone``` dentro do diretório ```/var/named```, conforme
descrito no arquivo ```named.conf```:

```shell
cat > /var/named/masterlab.com.zone <<'EOF'
$TTL 1D
@   IN  SOA dns.masterlab.com. root.masterlab.com. (
        2025100801 ; Serial (AAAAMMDDNN)
        1D         ; Refresh
        1H         ; Retry
        1W         ; Expire
        3H )       ; Minimum

@   IN NS dns.masterlab.com.

; Hosts do Lab
dns     IN  A   10.10.10.10
web     IN  A   10.10.10.20
client  IN  A   10.10.10.30

; Conveniências
@       IN  A   10.10.10.10
www     IN  CNAME web
EOF
```

Valide se a configuração está correta:

```shell
named-checkzone masterlab.com /var/named/masterlab.com.zone
```

Crie o arquivo de zona para busca reversa no mesmo diretório, mas com o nome ```masterlab.com.rev.zone```.

```shell
cat > /var/named/masterlab.com.rev.zone <<'EOF'
$TTL 1D
@   IN  SOA dns.masterlab.com. root.masterlab.com. (
        2025100801 ; Serial
        1D         ; Refresh
        1H         ; Retry
        1W         ; Expire
        3H )       ; Minimum

@   IN NS dns.masterlab.com.

10  IN  PTR dns.masterlab.com.
20  IN  PTR web.masterlab.com.
30  IN  PTR client.masterlab.com.
EOF
```

Valide se a configuração está correta:

```shell
named-checkzone 10.10.10.in-addr.arpa /var/named/masterlab.com.rev.zone
```

Atualize as permissões nos arquivos criados:

```shell
chown root:named /var/named/masterlab.com.zone /var/named/masterlab.com.rev.zone
chmod 640 /var/named/masterlab.com.zone /var/named/masterlab.com.rev.zone
restorecon -v /var/named/masterlab.com.zone /var/named/masterlab.com.rev.zone
```

Reinicie o serviço:

```shell
systemctl restart named
systemctl status named --no-pager
ss -lntu | grep -E ':53'
```

O firewall já foi configurado no provisionamento, mas você pode conferir:

```shell
firewall-cmd --list-services | grep dns || true
```

Ao final, você deve configurar o seu cliente para resolver nomes nestes servidores. Para tornar permanente o apontamento, você pode editar o arquivo Netplan do Vagrant (geralmente ```/etc/netplan/50-vagrant.yaml```) adicionando nameservers: [10.10.10.10] e aplicando com sudo netplan apply. Veja exemplo:

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    # Adaptador NAT do Vagrant: mantém DHCP para acesso à internet/updates
    enp0s3:
      dhcp4: true

    # Adaptador de rede privada do lab
    enp0s8:
      dhcp4: false
      addresses:
        - 10.10.10.30/24
      # Sem gateway aqui (o default gateway fica no NAT enp0s3)
      nameservers:
        search:
          - masterlab.com
        addresses:
          - 10.10.10.10
```

Aplique e valide:

```shell
sudo netplan apply
dig @10.10.10.10 masterlab.com NS +norec
dig @10.10.10.10 dns.masterlab.com A +norec
dig -x 10.10.10.20 @10.10.10.10 +norec
```