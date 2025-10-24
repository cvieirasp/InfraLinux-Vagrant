# Configuração do Servidor LDAP

## 1. Configuração do DNS

Acessar o servidor DNS via SSH:

```shell
vagrant ssh dns
sudo -i
```

Editar o arquivo da zona direta ```/var/named/masterlab.com.zone``` para adicionar o registro A do servidor de email:

```shell
vim /var/named/masterlab.com.zone
```

Adicionar a linha do servidor de email na seção de registros A (mantendo a sequência dos IPs), não esquecendo de incrementar o serial number para que as mudanças sejam propagadas:

```shell
$TTL 1D
@   IN  SOA dns.masterlab.com. root.masterlab.com. (
        2025100804 ; Serial (AAAAMMDDNN)
        1D         ; Refresh
        1H         ; Retry
        1W         ; Expire
        3H )       ; Minimum

@ IN NS dns.masterlab.com.

; Hosts do lab
dns     IN  A   10.10.10.10
web     IN  A   10.10.10.20
client  IN  A   10.10.10.30
nginx   IN  A   10.10.10.40
mail    IN  A   10.10.10.50
ldap    IN  A   10.10.10.60

; Vhosts (apontamento para o proxy nginx)
web.financeiro  IN A 10.10.10.40
web.rh          IN A 10.10.10.40

; Conveniências
@               IN A     10.10.10.10
www             IN CNAME web
www.financeiro  IN CNAME web.financeiro.masterlab.com.
www.rh          IN CNAME web.rh.masterlab.com.
```

Editar o arquivo da zona reversa ```/var/named/masterlab.com.rev.zone``` para adicionar o registro A do servidor de email:

```shell
vim /var/named/masterlab.com.rev.zone
```

Adicionar a linha do servidor de email na seção de registros PTR, não esquecendo de incrementar o serial number para que as mudanças sejam propagadas:

```shell
$TTL 1D
@   IN  SOA dns.masterlab.com. root.masterlab.com. (
        2025100804 ; Serial
        1D         ; Refresh
        1H         ; Retry
        1W         ; Expire
        1D )       ; Minimum

@ IN NS dns.masterlab.com.

10  IN  PTR dns.masterlab.com.
20  IN  PTR web.masterlab.com.
30  IN  PTR client.masterlab.com.
40  IN  PTR nginx.masterlab.com.
50  IN  PTR mail.masterlab.com.
60  IN  PTR ldap.masterlab.com.
```

Verificar a sintaxe da s zonas

```shell
named-checkzone masterlab.com /var/named/masterlab.com.zone
named-checkzone 10.10.10.in-addr.arpa /var/named/masterlab.com.rev.zone
```

Reiniciar o serviço e verificar seu status:

```shell
systemctl restart named
systemctl status named --no-pager
ss -lntu | grep -E ':53'
```

Testar a resolução do próprio servidor DNS:

```shell
dig @localhost ldap.masterlab.com
dig @localhost -x 10.10.10.60
```

Testar a resolução do cliente:

```shell
vagrant ssh client
dig @10.10.10.10 mail.masterlab.com
dig @10.10.10.10 -x 10.10.10.60
nslookup mail.masterlab.com 10.10.10.60
```

