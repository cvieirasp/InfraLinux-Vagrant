# Configuração do Servidor de Email

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
        2025100803 ; Serial (AAAAMMDDNN)
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
        2025100803 ; Serial
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
dig @localhost mail.masterlab.com
dig @localhost -x 10.10.10.50
```

Testar a resolução do cliente:

```shell
vagrant ssh client
dig @10.10.10.10 mail.masterlab.com
dig @10.10.10.10 -x 10.10.10.50
nslookup mail.masterlab.com 10.10.10.10
```

## 2. Preparação do Ambiente

Acessar o servidor via SSH:

```shell
vagrant ssh mail
sudo -i
```

Gerar certificados SSL

```shell
openssl genrsa -out /etc/pki/tls/private/mail.masterlab.com.key 2048
openssl req -new -x509 -key /etc/pki/tls/private/mail.masterlab.com.key \
    -out /etc/pki/tls/certs/mail.masterlab.com.crt -days 365 \
    -subj "/C=BR/ST=MG/L=BeloHorizonte/O=MasterLab/OU=IT/CN=mail.masterlab.com"
chmod 600 /etc/pki/tls/private/mail.masterlab.com.key
chmod 644 /etc/pki/tls/certs/mail.masterlab.com.crt

# Ajustar permissões
chmod 600 /etc/pki/tls/private/mail.masterlab.com.key
chmod 644 /etc/pki/tls/certs/mail.masterlab.com.crt
chown root:root /etc/pki/tls/private/mail.masterlab.com.key
chown root:root /etc/pki/tls/certs/mail.masterlab.com.crt
```

## 3. Configuração do Postfix

Configurar Postfix para Maildir

```shell
# Configurações básicas
postconf -e 'myhostname = mail.masterlab.com'
postconf -e 'mydomain = masterlab.com'
postconf -e 'myorigin = $mydomain'
postconf -e 'inet_interfaces = all'
postconf -e 'inet_protocols = ipv4'
postconf -e 'mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain'
postconf -e 'mynetworks = 127.0.0.0/8, 10.10.10.0/24'

# Configurar para usar Maildir
postconf -e 'home_mailbox = Maildir/'

# Configurações TLS/SSL
postconf -e 'smtpd_tls_cert_file = /etc/pki/tls/certs/mail.masterlab.com.crt'
postconf -e 'smtpd_tls_key_file = /etc/pki/tls/private/mail.masterlab.com.key'
postconf -e 'smtpd_tls_security_level = may'
postconf -e 'smtpd_tls_auth_only = yes'
postconf -e 'smtpd_tls_received_header = yes'
postconf -e 'smtpd_tls_session_cache_timeout = 3600s'

# Configurações SASL
postconf -e 'smtpd_sasl_type = dovecot'
postconf -e 'smtpd_sasl_path = private/auth'
postconf -e 'smtpd_sasl_auth_enable = yes'
postconf -e 'smtpd_sasl_security_options = noanonymous'
postconf -e 'smtpd_sasl_local_domain = $myhostname'
postconf -e 'broken_sasl_auth_clients = yes'

# Restrições de segurança
postconf -e 'smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination'
postconf -e 'smtpd_sender_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unknown_sender_domain'
```

Editar o arquivo ```master.cf``` para configurar a porta 587 (Submission):

```shell
vim /etc/postfix/master.cf
```

Localizar e descomentar/modificar a linha da porta 587:

```shell
submission inet n       -       n       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

## 4. Configuração do Dovecot

Editar a configuração principal do Dovecot:

```shell
# Configuração principal do Dovecot
vim /etc/dovecot/dovecot.conf
```

Adicionar/modificar:

```shell
protocols = imap pop3 lmtp
listen = *, ::
```

Configurar Maildir no Dovecot:

```shell
vim /etc/dovecot/conf.d/10-mail.conf
```

Adicionar/modificar:

```shell
# CRÍTICO: Usar Maildir
mail_location = maildir:~/Maildir
mail_privileged_group = mail
```

Configurar SSL no Dovecot:

```shell
# SSL
vim /etc/dovecot/conf.d/10-ssl.conf
```

Adicionar/modificar:

```shell
ssl = required
ssl_cert = </etc/pki/tls/certs/mail.masterlab.com.crt
ssl_key = </etc/pki/tls/private/mail.masterlab.com.key
ssl_protocols = !SSLv2 !SSLv3
```

Configurar autenticação:

```shell
# Autenticação
vim /etc/dovecot/conf.d/10-auth.conf
```

Adicionar/modificar:

```shell
auth_mechanisms = plain login
disable_plaintext_auth = yes
```

Configurar SASL para integração com Postfix:

```shell
vim /etc/dovecot/conf.d/10-master.conf
```

Localizar e modificar a seção de autenticação:

```shell
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
}
```

## 5. Criação de Usuários e Verificação

Criar Usuários com Maildir

```shell
# Criar usuários
for user in financeiro rh; do
    useradd -m -s /bin/bash $user
    echo "$user:password123" | chpasswd
    
    # Criar estrutura Maildir
    mkdir -p /home/$user/Maildir/{cur,new,tmp}
    chown -R $user:$user /home/$user/Maildir
    chmod -R 700 /home/$user/Maildir
done
```

Verificar e Reiniciar serviços:

```shell
# Validar configurações
postfix check
doveconf -n

# Verificar status
systemctl status postfix --no-pager
systemctl status dovecot --no-pager

# Verificar portas
ss -tlnp | grep -E ':587|:993|:995|:143|:110'
```

## 6. Teste de Envio de Email no Cliente

Criar e executar o script para facilitar os testes:

```shell
#!/bin/bash
# filepath: /tmp/test_email.sh

# Codificar credenciais em base64
USER_B64=$(echo -n "financeiro" | base64)
PASS_B64=$(echo -n "password123" | base64)

echo "Testando envio de email..."
echo "Usuário (base64): $USER_B64"
echo "Senha (base64): $PASS_B64"

# Criar arquivo temporário com comandos SMTP
cat > /tmp/smtp_commands << EOF
EHLO client.masterlab.com
AUTH LOGIN
$USER_B64
$PASS_B64
MAIL FROM:<financeiro@masterlab.com>
RCPT TO:<rh@masterlab.com>
DATA
Subject: Teste automatizado
From: financeiro@masterlab.com
To: rh@masterlab.com
Date: $(date)

Esta é uma mensagem de teste automatizada.
Enviada pelo Financeiro para RH.
.
QUIT
EOF

# Executar teste
openssl s_client -starttls smtp -ign_eof -crlf -connect mail.masterlab.com:587 < /tmp/smtp_commands

# Limpar arquivo temporário
rm /tmp/smtp_commands
```

Verificar se o RH consegue ler seus emails:

```shell
# Conectar via IMAP SSL
openssl s_client -connect mail.masterlab.com:993

# Depois da conexão SSL estabelecida:
a001 LOGIN rh password123
a002 LIST "" "*"
a003 SELECT INBOX
a004 FETCH 1 FULL
a005 LOGOUT
```

## 7. Verificar Recebimento no Servidor

Verificar os arquivos na estrutura criada:

```shell
# Verificar estrutura Maildir
ls -la /home/rh/Maildir/
ls -la /home/rh/Maildir/new/

# Ler email (cada arquivo é um email)
cat /home/rh/Maildir/new/*
```

[Voltar](/)