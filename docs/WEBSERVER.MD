# Configuração do Servidor WEB

## 1. Preparação do ambiente

Acessar o servidor via SSH:

```shell
vagrant ssh web
sudo -i
```

Criar as pastas onde serão hospedadas as páginas da aplicação e logs:

```shell
mkdir -p /var/www/sites/web.financeiro.masterlab.com/public_html
mkdir -p /var/www/sites/web.masterlab.com/logs
mkdir -p /var/www/sites/web.financeiro.masterlab.com/public_html
mkdir -p /var/www/sites/web.masterlab.com/logs
```

> Serão criados dois vhosts, financeiro e rh.

Criação da página do Financeiro:

```shell
cat > /var/www/sites/web.financeiro.masterlab.com/public_html/index.html <<'EOF'
<!doctype html>
<html lang="pt-br">
<head>
  <meta charset="utf-8">
  <title>Portal Financeiro — masterlab</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <style>
    :root { --bg:#0f172a; --card:#111827; --text:#e5e7eb; --muted:#94a3b8; }
    * { box-sizing: border-box; }
    body { margin:0; font-family: system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Arial,sans-serif; background: linear-gradient(120deg,#0b1220,#0f172a); color:var(--text); }
    .wrap { min-height:100vh; display:flex; align-items:center; justify-content:center; padding:24px; }
    .card { width:100%; max-width:720px; background:var(--card); border:1px solid rgba(255,255,255,.06); border-radius:16px; padding:28px; box-shadow:0 10px 30px rgba(0,0,0,.25); }
    h1 { margin:0 0 8px; font-size:28px; }
    p  { margin:8px 0; line-height:1.6; color:var(--muted); }
    .pill { display:inline-block; padding:6px 10px; border-radius:999px; background:#1f2937; font-size:12px; color:#cbd5e1; }
    .grid { margin-top:16px; display:grid; gap:12px; grid-template-columns: repeat(auto-fit,minmax(180px,1fr)); }
    .kpi { padding:14px; border:1px solid rgba(255,255,255,.08); border-radius:12px; background:#0b1220; }
    .kpi b { display:block; font-size:20px; color:#f1f5f9; }
    footer { margin-top:20px; font-size:12px; color:#94a3b8; }
    code { background:#0b1220; padding:2px 6px; border-radius:6px; color:#e5e7eb; }
  </style>
</head>
<body>
  <div class="wrap">
    <main class="card">
      <span class="pill">web.financeiro.masterlab.com</span>
      <h1>Portal Financeiro</h1>
      <p>Este site está rodando no servidor Apache da VM <code>web</code> (IP 10.10.10.20). Use esta página como ponto de partida para seus testes.</p>

      <div class="grid">
        <div class="kpi"><span>Receitas (demo)</span><b>R$ 120.000</b></div>
        <div class="kpi"><span>Despesas (demo)</span><b>R$ 85.000</b></div>
        <div class="kpi"><span>Resultado (demo)</span><b>R$ 35.000</b></div>
      </div>

      <footer>
        Ambiente de laboratório • Masterlab • Apache HTTPD • Rocky Linux 9
      </footer>
    </main>
  </div>
</body>
</html>
EOF
```

Criação da página do RH:

```shell
cat > /var/www/sites/web.rh.masterlab.com/public_html/index.html <<'EOF'
<!doctype html>
<html lang="pt-br">
<head>
  <meta charset="utf-8">
  <title>Portal de RH — masterlab</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <style>
    :root { --bg:#031b1a; --card:#05201f; --text:#e6fffb; --muted:#a7f3d0; }
    * { box-sizing: border-box; }
    body { margin:0; font-family: system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Arial,sans-serif; background: radial-gradient(1200px 600px at 10% 20%,#06302e,transparent), linear-gradient(160deg,#031b1a,#051b2f); color:var(--text); }
    .wrap { min-height:100vh; display:flex; align-items:center; justify-content:center; padding:24px; }
    .card { width:100%; max-width:720px; background:var(--card); border:1px solid rgba(255,255,255,.08); border-radius:16px; padding:28px; box-shadow:0 10px 30px rgba(0,0,0,.25); }
    h1 { margin:0 0 8px; font-size:28px; }
    p  { margin:8px 0; line-height:1.6; color:var(--muted); }
    .pill { display:inline-block; padding:6px 10px; border-radius:999px; background:#06302e; font-size:12px; color:#ccfbf1; }
    .list { margin-top:12px; padding-left:18px; }
    .list li { margin:6px 0; }
    footer { margin-top:20px; font-size:12px; color:#99f6e4; }
    code { background:#06302e; padding:2px 6px; border-radius:6px; color:#e6fffb; }
  </style>
</head>
<body>
  <div class="wrap">
    <main class="card">
      <span class="pill">web.rh.masterlab.com</span>
      <h1>Portal de RH</h1>
      <p>Site de laboratório para o domínio <code>web.rh.masterlab.com</code> rodando no Apache (VM <code>web</code>, IP 10.10.10.20).</p>
      <ul class="list">
        <li>Vagas internas (demo)</li>
        <li>Treinamentos (demo)</li>
        <li>Políticas e documentos (demo)</li>
      </ul>
      <footer>
        Ambiente de laboratório • Masterlab • Apache HTTPD • Rocky Linux 9
      </footer>
    </main>
  </div>
</body>
</html>
EOF
```
Aplicar Contextos/Permissões (SELinux + donos):

```shell
chown -R apache:apache /var/www/sites/web.financeiro.masterlab.com
# Cria uma regra persistente de rotulagem (file context) no SELinux dizendo que o caminho em questão e tudo dentro dele deve ter tipo httpd_sys_content_t.
semanage fcontext -a -t httpd_sys_content_t '/var/www/sites/web.financeiro.masterlab.com(/.*)?'
# Adiciona regra persistente dizendo que o diretório de logs do vhost e seus conteúdos devem ter tipo httpd_log_t.
semanage fcontext -a -t httpd_log_t '/var/www/sites/web.financeiro.masterlab.com/logs(/.*)?'
# Aplica no disco os rótulos (contexts) de SELinux conforme as regras conhecidas — incluindo as que você acabou de criar com semanage fcontext.
restorecon -Rv /var/www/sites/web.financeiro.masterlab.com

chown -R apache:apache /var/www/sites/web.rh.masterlab.com
semanage fcontext -a -t httpd_sys_content_t '/var/www/sites/web.rh.masterlab.com(/.*)?'
semanage fcontext -a -t httpd_log_t '/var/www/sites/web.rh.masterlab.com/logs(/.*)?'
restorecon -Rv /var/www/sites/web.rh.masterlab.com
```

> Se ```semanage``` não existir: ```dnf -y install policycoreutils-python-utils```.

## 2. Virtual Host

Criar o arquivo do vhost (Financeiro) em ```/etc/httpd/conf.d/web.financeiro.masterlab.com.conf```:

```shell
cat > /etc/httpd/conf.d/web.financeiro.masterlab.com.conf <<'EOF'
# VirtualHost simples HTTP para web.financeiro.masterlab.com
<VirtualHost *:80>
    ServerName web.financeiro.masterlab.com
    ServerAlias web.financeiro.masterlab.com

    DocumentRoot /var/www/sites/web.financeiro.masterlab.com/public_html
    ErrorLog   /var/www/sites/web.financeiro.masterlab.com/logs/error.log
    CustomLog  /var/www/sites/web.financeiro.masterlab.com/logs/access.log combined

    <Directory /var/www/sites/web.financeiro.masterlab.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>

# Opcional: página padrão se acessar por IP/sem Host header
#_default_:80 já é tratado pelo vhost principal do httpd; manter assim é OK.
EOF
```

Criar o arquivo do vhost (RH) em ```/etc/httpd/conf.d/web.rh.masterlab.com.conf```:

```shell
cat > /etc/httpd/conf.d/web.rh.masterlab.com.conf <<'EOF'
# VirtualHost simples HTTP para web.rh.masterlab.com
<VirtualHost *:80>
    ServerName web.rh.masterlab.com
    ServerAlias web.rh.masterlab.com

    DocumentRoot /var/www/sites/web.rh.masterlab.com/public_html
    ErrorLog   /var/www/sites/web.rh.masterlab.com/logs/error.log
    CustomLog  /var/www/sites/web.rh.masterlab.com/logs/access.log combined

    <Directory /var/www/sites/web.rh.masterlab.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>

# Opcional: página padrão se acessar por IP/sem Host header
#_default_:80 já é tratado pelo vhost principal do httpd; manter assim é OK.
EOF
```

Validar a sintaxe:

```shell
httpd -t
```

## 4. Status do Serviço

Reiniciar o serviço e verificar seu status:

```shell
systemctl reload httpd   # Ou restart para garantir
systemctl status httpd --no-pager
```

O firewall já foi configurado no provisionamento, mas pode ser conferido e habilitado utilizando o comando abaixo:

```shell
firewall-cmd --list-services | grep http || (firewall-cmd --permanent --add-service=http && firewall-cmd --reload)
```

## 5. Testes no Cliente

No cliente, se o DNS autoritativo já foi configurado corretamente, basta:

```shell
curl -I http://web.financeiro.masterlab.com
curl -I http://web.rh.masterlab.com
```

Para abrir o HTML:

```shell
curl http://web.financeiro.masterlab.com | head
curl http://web.rh.masterlab.com | head
```

[Voltar](../README.md)