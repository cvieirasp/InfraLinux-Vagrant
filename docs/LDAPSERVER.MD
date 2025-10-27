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
dig @10.10.10.10 ldap.masterlab.com
dig @10.10.10.10 -x 10.10.10.60
nslookup ldap.masterlab.com 10.10.10.60
```

## 2. Configuração Inicial do Servidor LDAP

Acessar o servidor LDAP via SSH:

```shell
vagrant ssh ldap
sudo -i
```

Para que possamos inserir qualquer dado (entry) no LDAP, precisamos de ObjectClasses para que possamos usar os
atributos. E para ter as ObjectClasses, precisamos importar elas por meio de schemas.
Acho mais fácil já fazer tudo como root:

```shell
ldapadd -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
ldapadd -H ldapi:/// -f /etc/openldap/schema/nis.ldif
```

Configurar Senha do Administrador e gerar hash da senha:

```shell
slappasswd
```

*Digite uma senha e anote o hash gerado*

Criar arquivo de configuração para o domínio:

```shell
vim /etc/openldap/domain.ldif
```

Conteúdo do arquivo:

```ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=masterlab,dc=com

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=masterlab,dc=com

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}SEU_HASH_AQUI
```

*Substitua SEU_HASH_AQUI pelo hash gerado no passo anterior*

Aplicar a configuração:

```shell
ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/openldap/domain.ldif
```

Criar arquivo de ACL:

```shell
vim /etc/openldap/acl.ldif
```

Conteúdo do arquivo:

```ldif
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=admin,dc=masterlab,dc=com"

dn: olcDatabase={0}config,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage by dn.base="cn=admin,dc=masterlab,dc=com"

dn: olcDatabase={-1}frontend,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to attrs=userPassword by self write by anonymous auth by * none

dn: olcDatabase={-1}frontend,cn=config
changetype: modify
add: olcAccess
olcAccess: {1}to dn.subtree="ou=people,dc=masterlab,dc=com" by dn.base="cn=admins,ou=groups,dc=masterlab,dc=com" read

dn: olcDatabase={-1}frontend,cn=config
changetype: modify
add: olcAccess
olcAccess: {2}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage by dn.base="cn=admin,dc=masterlab,dc=com"
```

Aplicar a configuração:

```shell
ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/openldap/acl.ldif
```

Criar arquivo com a estrutura base:

```shell
vim /etc/openldap/base.ldif
```

Conteúdo do arquivo:

```ldif
dn: dc=masterlab,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: MasterLab Organization
dc: masterlab

dn: cn=admin,dc=masterlab,dc=com
objectClass: organizationalRole
cn: admin
description: LDAP Administrator

dn: ou=people,dc=masterlab,dc=com
objectClass: organizationalUnit
ou: people
description: All people in organization

dn: ou=groups,dc=masterlab,dc=com
objectClass: organizationalUnit
ou: groups
description: All groups in organization
```

Adicionar a estrutura base:

```shell
ldapadd -x -D cn=admin,dc=masterlab,dc=com -W -f /etc/openldap/base.ldif
```

*Digite a senha definida anteriormente*

Criar arquivo com usuários de exemplo:

```shell
vim /etc/openldap/users.ldif
```

```ldif
dn: uid=john,ou=people,dc=masterlab,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: john
sn: Doe
givenName: John
cn: John Doe
displayName: John Doe
uidNumber: 1001
gidNumber: 1001
userPassword: {crypt}x
gecos: John Doe
loginShell: /bin/bash
homeDirectory: /home/john
mail: john.doe@masterlab.com

dn: uid=jane,ou=people,dc=masterlab,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: jane
sn: Doe
givenName: Jane
cn: Jane Doe
displayName: Jane Doe
uidNumber: 1001
gidNumber: 1001
userPassword: {crypt}x
gecos: Jane Doe
loginShell: /bin/bash
homeDirectory: /home/jane
mail: jane.doe@masterlab.com
```

Adicionar os usuários:

```shell
ldapadd -x -D cn=admin,dc=masterlab,dc=com -W -f /etc/openldap/users.ldif
```

Definir senhas para os usuários:

```shell
# Para o usuário John
ldappasswd -S -W -D "cn=admin,dc=masterlab,dc=com" -x "uid=john,ou=people,dc=masterlab,dc=com"
# Para a usuária Jane
ldappasswd -S -W -D "cn=admin,dc=masterlab,dc=com" -x "uid=jane,ou=people,dc=masterlab,dc=com"
```

Criar arquivo com grupos:

```shell
vim /etc/openldap/groups.ldif
```

Conteúdo do arquivo:

```ldif
dn: cn=admins,ou=groups,dc=masterlab,dc=com
objectClass: groupOfNames
cn: admins
description: System Administrators
member: uid=jane,ou=people,dc=masterlab,dc=com

dn: cn=developers,ou=groups,dc=masterlab,dc=com
objectClass: groupOfNames
cn: developers
description: Development Team
member: uid=john,ou=people,dc=masterlab,dc=com
member: uid=jane,ou=people,dc=masterlab,dc=com
```

Adicionar os grupos:

```shell
ldapadd -x -D cn=admin,dc=masterlab,dc=com -W -f /etc/openldap/groups.ldif
```

Habilitar Log de Auditoria:

```shell
vim /etc/openldap/logging.ldif
```

Conteúdo do arquivo:

```ldif
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats
```

Aplicar modificação:

```shell
ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/openldap/logging.ldif
systemctl restart slapd
```

Configurar Índices para Performance:

```shell
vim /etc/openldap/indexes.ldif
```

Conteúdo do arquivo:

```ldif
dn: olcDatabase=mdb,cn=config
changetype: modify
add: olcDbIndex
olcDbIndex: uid eq,pres,sub
olcDbIndex: mail eq,pres,sub
olcDbIndex: cn eq,pres,sub
olcDbIndex: memberUid eq,pres,sub
```

Aplicar modificação:

```shell
ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/openldap/indexes.ldif
systemctl restart slapd
```

## 3. Testes e Verificações

Testar Conectividade LDAP:

```shell
# Testar conexão local
ldapsearch -x -H ldap://localhost -b "dc=masterlab,dc=com"

# Testar busca por usuários
ldapsearch -x -H ldap://localhost -b "ou=people,dc=masterlab,dc=com"

# Testar busca por grupos
ldapsearch -x -H ldap://localhost -b "ou=groups,dc=masterlab,dc=com"

# Testar autenticação de usuário
ldapwhoami -x -D "uid=john,ou=people,dc=masterlab,dc=com" -W
ldapwhoami -x -D "uid=jane,ou=people,dc=masterlab,dc=com" -W
```

Verificar Logs

```shell
# Verificar logs do slapd
journalctl -u slapd --no-pager -l

# Verificar logs em tempo real
journalctl -u slapd -f
```

## 4. Testes do Cliente

Instalar ferramentas:

- **libnss-ldap:** Biblioteca que permite ao sistema buscar informações de usuários e grupos (passwd, group etc.) no LDAP em vez de só nos arquivos locais tipo /etc/passwd.
- **libpam-ldap:** Módulo PAM que permite validar a senha de um usuário contra o LDAP na hora do login, su, ssh etc.
- **ldap-utils:** Conjunto de ferramentas de linha de comando (como ldapsearch, ldapadd, ldapmodify, ldapwhoami) para consultar e administrar um servidor LDAP manualmente.
- **nscd (Name Service Cache Daemon):** Serviço que faz cache local de informações de nomes de usuários, grupos, hosts etc., para evitar buscar essas infos toda hora no LDAP (ou DNS, NIS etc.) e melhorar desempenho.

```shell
# Instalar cliente LDAP 
sudo apt-get update
sudo apt-get install -y libnss-ldap libpam-ldap ldap-utils nscd
```

Durante a instalação, configure:

- LDAP server URI: **ldap://10.10.10.60**
- Distinguished name of the search base: **dc=masterlab,dc=com**
- LDAP version: **3**
- Make local root Database admin: **Yes**
- Does the LDAP database require login: **No**
- LDAP account for root: **cn=admin,dc=masterlab,dc=com**
- LDAP root account password: **(senha do admin definida)**

Configurar NSS (Name Service Switch):

Editar o arquivo ```/etc/nsswitch.conf```:

```shell
sudo vim /etc/nsswitch.conf
```

Modificar as linhas para incluir ```ldap```:

```
passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files ldap
```

Editar a configuração de sessão:

```shell
sudo vim /etc/nsswitch.conf
```

Adicionar:

```
session required        pam_mkhomedir.so skel=/etc/skel umask=077
```

Testar a conectividade:

```shell
# Testar conexão remota
ldapsearch -x -H ldap://ldap.masterlab.com -b "dc=masterlab,dc=com"

# Testar autenticação remota
ldapwhoami -x -H ldap://ldap.masterlab.com -D "uid=john,ou=people,dc=masterlab,dc=com" -W
ldapwhoami -x -H ldap://ldap.masterlab.com -D "uid=jane,ou=people,dc=masterlab,dc=com" -W

# Verificar se os usuários LDAP são visíveis
getent passwd john
getent passwd jane

# Verificar grupos
getent group

# Verificar informações específicas
id john
id jane

# Testar trocar de usuário
sudo su - john
sudo su - jane
```
