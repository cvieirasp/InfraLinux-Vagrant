# Configuração do Servidor DNS

## 1. Preparação do ambiente

Acessar o servidor via SSH:

```shell
vagrant ssh dns
sudo -i
```

Fazer o backup do config padrão (boa prática):

```shell
cp -a /etc/named.conf /etc/named.conf.bak.$(date +%F-%H%M%S)
```

Criar as pastas de logs já com as devidas permissões:

```shell
mkdir /var/log/named
chgrp named /var/log/named
chmod 775 /var/log/named
```

## 2. Arquivo de Configuração

Editar o arquivo: ```named.conf```

```shell
vim /etc/named.conf
```

Utilizar o conteúdo abaixo:

```
acl "lab-net" { 127.0.0.1; 10.10.10.0/24; };

options {
  listen-on port 53 { 127.0.0.1; 10.10.10.10; };
  listen-on-v6 { none; };

  // ---- Servidor DNS Autoritativo puro ----
  recursion no; // Desativa a recursão
  allow-recursion { none; }; // Nenhum host com permissão de realizar recursão
  allow-query { 10.10.10.0/24 }; // Adiciona a rede para consulta das Zonas
  allow-query-cache { none; }; // Não responde Cache

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

Utilizar o comando abaixo para validar se a configuração está correta:

```shell
named-checkconf
```

## 3. Arquivos de Zonas

Criar o arquivo de zona chamado ```masterlab.com.zone``` dentro do diretório ```/var/named```, conforme
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

; Vhosts (mesmo IP do servidor web)
web.financeiro    IN  A   10.10.10.20
web.rh            IN  A   10.10.10.20

; Conveniências
@                 IN  A      10.10.10.10
www.financeiro    IN  CNAME  web.financeiro.masterlab.com.
www.rh            IN  CNAME  web.rh.masterlab.com.
EOF
```

Utilizar o comando abaixo para validar se a configuração está correta:

```shell
named-checkzone masterlab.com /var/named/masterlab.com.zone
```

Criar o arquivo de zona para busca reversa no mesmo diretório, mas com o nome ```masterlab.com.rev.zone```.

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

Utilizar o comando abaixo para validar se a configuração está correta:

```shell
named-checkzone 10.10.10.in-addr.arpa /var/named/masterlab.com.rev.zone
```

Atualizar as permissões nos arquivos criados:

```shell
chown root:named /var/named/masterlab.com.zone /var/named/masterlab.com.rev.zone
chmod 640 /var/named/masterlab.com.zone /var/named/masterlab.com.rev.zone
restorecon -v /var/named/masterlab.com.zone /var/named/masterlab.com.rev.zone
```
## 4. Status do Serviço

Reiniciar o serviço e verificar seu status:

```shell
systemctl restart named
systemctl status named --no-pager
ss -lntu | grep -E ':53'
```

O firewall já foi configurado no provisionamento, mas pode ser conferido e habilitado utilizando o comando abaixo:

```shell
firewall-cmd --list-services | grep dns || (firewall-cmd --permanent --add-service=dns && firewall-cmd --reload)
```

 ## 5. Testes no Cliente

Ao final, o cliente deve ser configurado para resolver nomes nestes servidores. Para tornar permanente o apontamento, é possível editar o arquivo **Netplan do Vagrant** (geralmente ```/etc/netplan/50-vagrant.yaml```) adicionando nameservers: ```[10.10.10.10]``` e aplicando com ```sudo netplan apply```. Veja exemplo:

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

Aplicar e validar:

```shell
sudo netplan apply
dig @10.10.10.10 masterlab.com NS +norec
dig @10.10.10.10 dns.masterlab.com A +norec
dig -x 10.10.10.20 @10.10.10.10 +norec
```

[Voltar](../README.md)