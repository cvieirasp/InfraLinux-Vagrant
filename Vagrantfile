VAGRANT_REQUIRE_PLUGINS = [] # sem plugins obrigatórios

Vagrant.configure("2") do |config|
  # Evita sincronização padrão da pasta (mais rápido/limpo para laboratório)
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # ====== DNS: Rocky Linux 9 + BIND (named) ======
  config.vm.define "dns" do |dns|
    dns.vm.box = "generic/rocky9"
    dns.vm.hostname = "dns.lab.local"
    dns.vm.network "private_network", ip: "10.10.10.10"

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
    web.vm.box = "generic/rocky9"
    web.vm.hostname = "web.lab.local"
    web.vm.network "private_network", ip: "10.10.10.20"

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
    client.vm.network "private_network", ip: "10.10.10.30"

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
end