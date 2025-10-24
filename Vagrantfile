VAGRANT_REQUIRE_PLUGINS = [] # sem plugins obrigatórios

Vagrant.configure("2") do |config|
  # Evita sincronização padrão da pasta (mais rápido/limpo para laboratório)
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # ====== DNS: Rocky Linux 9 + BIND (named) ======
  config.vm.define "dns" do |dns|
    dns.vm.box = "rockylinux/9"
    dns.vm.hostname = "dns.lab.local"
    dns.vm.network "private_network", ip: "10.10.10.10", guest: 22, host: 2222, id: "ssh", auto_correct: true

    dns.vm.provider "virtualbox" do |vb|
      vb.name = "lab-dns"
      vb.cpus = 1
      vb.memory = 1024
    end

    dns.vm.provision "shell", inline: <<-'SHELL'
      set -euxo pipefail
      # Atualiza metadados e instala BIND + utilitários + firewall
      sudo dnf -y install bind bind-utils firewalld

      # Habilita/ativa firewall e libera portas do DNS
      sudo systemctl enable --now firewalld
      sudo firewall-cmd --permanent --add-service=dns
      sudo firewall-cmd --reload

      # Habilita e inicia o serviço named (sem configurar zonas ainda)
      sudo systemctl enable --now named

      # Mensagem de estado
      echo "===== STATUS: named ====="
      systemctl --no-pager --full status named || true
    SHELL
  end

  # ====== WEB: Rocky Linux 9 + Apache (httpd) ======
  config.vm.define "web" do |web|
    web.vm.box = "rockylinux/9"
    web.vm.hostname = "web.lab.local"
    web.vm.network "private_network", ip: "10.10.10.20", host: 2223, id: "ssh", auto_correct: true

    web.vm.provider "virtualbox" do |vb|
      vb.name = "lab-web"
      vb.cpus = 1
      vb.memory = 1024
    end

    web.vm.provision "shell", inline: <<-'SHELL'
      set -euxo pipefail
      # Instala Apache + firewall
      sudo dnf -y install httpd firewalld

      # Habilita/ativa firewall e libera HTTP
      sudo systemctl enable --now firewalld
      sudo firewall-cmd --permanent --add-service=http
      sudo firewall-cmd --reload

      # Habilita e inicia Apache (sem configurar vhosts/pages custom)
      sudo systemctl enable --now httpd

      # Mensagem de estado
      echo "===== STATUS: httpd ====="
      systemctl --no-pager --full status httpd || true
    SHELL
  end

  # ====== CLIENT: Ubuntu 22.04 ======
  config.vm.define "client" do |client|
    client.vm.box = "ubuntu/jammy64"
    client.vm.hostname = "client.lab.local"
    client.vm.network "private_network", ip: "10.10.10.30", host: 2223, id: "ssh", auto_correct: true

    client.vm.provider "virtualbox" do |vb|
      vb.name = "lab-client"
      vb.cpus = 1
      vb.memory = 1024
    end

    client.vm.provision "shell", inline: <<-'SHELL'
      set -euxo pipefail
      sudo apt-get update -y
      # Utilitários para testes: dig, nslookup, curl, net-tools etc.
      sudo apt-get install -y dnsutils curl net-tools iproute2 traceroute
      # Nada de mexer em resolv.conf ainda; você fará a apontar pro DNS depois, manualmente.
      echo "===== CLIENT READY ====="
    SHELL
  end

  # ====== NGINX: Rocky Linux 9 + Nginx ======
  config.vm.define "nginx" do |nginx|
    nginx.vm.box = "rockylinux/9"
    nginx.vm.hostname = "nginx.lab.local"
    nginx.vm.network "private_network", ip: "10.10.10.40", host: 2224, id: "ssh", auto_correct: true

    nginx.vm.provider "virtualbox" do |vb|
      vb.name = "lab-nginx"
      vb.cpus = 1
      vb.memory = 1024
    end

    # Provisionamento: instala e habilita o Nginx
    nginx.vm.provision "shell", inline: <<-SHELL
      sudo dnf -y update
      sudo dnf -y install nginx

      # Habilita/ativa firewall e libera HTTP
      sudo systemctl enable --now firewalld
      sudo firewall-cmd --permanent --add-service=http
      sudo firewall-cmd --reload

      sudo systemctl enable nginx
      sudo systemctl start nginx
      
      # Página simples para validar
      echo "OK - Nginx on $(hostname) ($(hostname -I))" | sudo tee /usr/share/nginx/html/index.html
    SHELL
  end

  # ====== MAIL: Rocky Linux 9 + Postfix + Dovecot ======
  config.vm.define "mail" do |mail|
    mail.vm.box = "rockylinux/9"
    mail.vm.hostname = "mail.lab.local"
    mail.vm.network "private_network", ip: "10.10.10.50", host: 2225, id: "ssh", auto_correct: true

    mail.vm.provider "virtualbox" do |vb|
      vb.name = "lab-mail"
      vb.cpus = 1
      vb.memory = 1024
    end

    mail.vm.provision "shell", inline: <<-'SHELL'
      set -euxo pipefail
      # Instala Postfix (SMTP) + Dovecot (IMAP)
      sudo dnf -y update
      sudo dnf -y install postfix dovecot

      # Habilita/ativa firewall e libera portas de email
      sudo systemctl enable --now firewalld
      # sudo firewall-cmd --permanent --add-service=smtp            # Porta 25
      sudo firewall-cmd --permanent --add-service=smtps           # Porta 465
      sudo firewall-cmd --permanent --add-service=smtp-submission # Porta 587
      # sudo firewall-cmd --permanent --add-service=imap            # Porta 143
      sudo firewall-cmd --permanent --add-service=imaps           # Porta 993
      # sudo firewall-cmd --permanent --add-service=pop3            # Porta 110
      sudo firewall-cmd --permanent --add-service=pop3s           # Porta 995
      sudo firewall-cmd --reload

      # Habilita e inicia Postfix (SMTP)
      sudo systemctl enable --now postfix

      # Habilita e inicia Dovecot (IMAP/POP3)
      sudo systemctl enable --now dovecot

      # Mensagem de estado
      echo "===== STATUS: postfix ====="
      systemctl --no-pager --full status postfix || true
      echo "===== STATUS: dovecot ====="
      systemctl --no-pager --full status dovecot || true
      
      # Verificar portas abertas
      echo "===== PORTAS DE EMAIL ====="
      # ss -tlnp | grep -E ':25|:587|:465|:143|:993|:110|:995' || echo "Nenhuma porta de email detectada ainda"
      ss -tlnp | grep -E ':587|:465|:993|:995' || echo "Nenhuma porta de email detectada ainda"
    SHELL
  end
end