# Configuração do Servidor Proxy Reverso

## 1. Configuração do DNS

Acessar o servidor DNS via SSH:

```shell
vagrant ssh dns
sudo -i
```

Editar o arquivo da zona direta ```/var/named/masterlab.com.zone``` para adicionar o registro A do nginx:

```shell
vim /var/named/masterlab.com.zone
```

Adicionar a linha do nginx na seção de registros A (mantendo a sequência dos IPs), não esquecendo de incrementar o serial number para que as mudanças sejam propagadas:

```shell
$TTL 1D
@   IN  SOA dns.masterlab.com. root.masterlab.com. (
        2025100802 ; Serial (AAAAMMDDNN)
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

; Vhosts (apontamento para o proxy nginx)
web.financeiro  IN A 10.10.10.40
web.rh          IN A 10.10.10.40

; Conveniências
@               IN A     10.10.10.10
www             IN CNAME web
www.financeiro  IN CNAME web.financeiro.masterlab.com.
www.rh          IN CNAME web.rh.masterlab.com.
```

Editar o arquivo da zona reversa ```/var/named/masterlab.com.rev.zone``` para adicionar o registro A do nginx:

```shell
vim /var/named/masterlab.com.rev.zone
```

Adicionar a linha do nginx na seção de registros PTR, não esquecendo de incrementar o serial number para que as mudanças sejam propagadas:

```shell
$TTL 1D
@   IN  SOA dns.masterlab.com. root.masterlab.com. (
        2025100802 ; Serial
        1D         ; Refresh
        1H         ; Retry
        1W         ; Expire
        1D )       ; Minimum

@ IN NS dns.masterlab.com.

10  IN  PTR dns.masterlab.com.
20  IN  PTR web.masterlab.com.
30  IN  PTR client.masterlab.com.
40  IN  PTR nginx.masterlab.com.
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
dig @localhost nginx.masterlab.com
dig @localhost -x 10.10.10.40
```

Testar a resolução do cliente:

```shell
vagrant ssh client
dig @10.10.10.10 nginx.masterlab.com
dig @10.10.10.10 -x 10.10.10.40
nslookup nginx.masterlab.com 10.10.10.10
```

Testar conectividade HTTP à partir do cliente:

```shell
curl -I http://nginx.masterlab.com
```

## 2. Preparação do ambiente

Acessar o servidor via SSH:

```shell
vagrant ssh nginx
sudo -i
```

 O Nginx não tem permissão para fazer conexões de rede para outros servidores. Para isso, verificar o Status do SELinux:

```shell
# Verificar se SELinux está ativo
sestatus
# Verificar configurações de rede do httpd/nginx
getsebool httpd_can_network_connect
```

Permitir conexões de rede do Nginx/HTTP:

```shell
# Permitir que o nginx faça conexões de rede
sudo setsebool -P httpd_can_network_connect on

# Verificar se a configuração foi aplicada
getsebool httpd_can_network_connect
```

Criar configuração para o domínio financeiro

```shell
cat > /etc/nginx/conf.d/web.financeiro.masterlab.com.conf <<'EOF'
server {
    listen 80;
    server_name web.financeiro.masterlab.com;
    
    # Log específico para este vhost
    access_log /var/log/nginx/web.financeiro.access.log;
    error_log /var/log/nginx/web.financeiro.error.log;
    
    # Configurações do proxy
    location / {
        proxy_pass http://web.masterlab.com;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
EOF
```

Criar configuração para o domínio RH

```shell
cat > /etc/nginx/conf.d/web.rh.masterlab.com.conf <<'EOF'
server {
    listen 80;
    server_name web.rh.masterlab.com;
    
    # Log específico para este vhost
    access_log /var/log/nginx/web.rh.access.log;
    error_log /var/log/nginx/web.rh.error.log;
    
    # Configurações do proxy
    location / {
        proxy_pass http://web.masterlab.com;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
EOF
```

Testar a configuração do Nginx

```shell
nginx -t
```

## 4. Status do Serviço

Reiniciar o serviço e verificar seu status:

```shell
systemctl reload nginx
systemctl status nginx --no-pager
```

O firewall já foi configurado no provisionamento, mas pode ser conferido e habilitado utilizando o comando abaixo:

```shell
firewall-cmd --list-services | grep http || (firewall-cmd --permanent --add-service=http && firewall-cmd --reload)
```

## 5. Bloquear requisições HTTP no Servidor WEB para clientes

Alterar Firewall para permitir HTTP apenas do Nginx:

```shell
# Remover a regra geral de HTTP
firewall-cmd --permanent --remove-service=http
# Adicionar regra específica permitindo apenas o IP do Nginx
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.10.40" service name="http" accept'
# Permitir acesso local para testes/manutenção
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="127.0.0.1" service name="http" accept'
# Aplicar as mudanças
firewall-cmd --reload
# Verificar as regras ativas
firewall-cmd --list-all
```

Testar e recarregar Apache

```shell
httpd -t
systemctl reload httpd
```

## 6. Testes no Cliente

No cliente, teste se o proxy está funcionando:

```shell
curl -I http://web.financeiro.masterlab.com
curl -I http://web.rh.masterlab.com
```

Para abrir o HTML:

```shell
curl http://web.financeiro.masterlab.com | head
curl http://web.rh.masterlab.com | head
```

[Voltar](/)