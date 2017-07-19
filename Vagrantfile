# -*- mode: ruby -*-
# vi: set ft=ruby :

# Install a kube cluster using kubeadm:
# http://kubernetes.io/docs/getting-started-guides/kubeadm/

# See README.md for relevant environment variables and configuration settings

require 'openssl'
require 'securerandom'
require 'uri'

def get_environment_variable(envvar)
  ENV[envvar] || ENV[envvar.upcase] || ENV[envvar.downcase]
end

def apply_vm_hardware_customizations(provider)
  provider.linked_clone = true

  provider.customize ["modifyvm", :id, "--vram", "16"]
  provider.customize ["modifyvm", :id, "--largepages", "on"]
  provider.customize ["modifyvm", :id, "--nestedpaging", "on"]
  provider.customize ["modifyvm", :id, "--vtxvpid", "on"]
  provider.customize ["modifyvm", :id, "--hwvirtex", "on"]
  provider.customize ["modifyvm", :id, "--ioapic", "on"]
  provider.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
  provider.customize ["modifyvm", :id, "--uart2", "0x2F8", "3"]
  provider.customize ["modifyvm", :id, "--uartmode1", "disconnected"]
  provider.customize ["modifyvm", :id, "--uartmode2", "disconnected"]
end

# hook extra disk creation in here
class VagrantPlugins::ProviderVirtualBox::Action::SetName
  alias_method :original_call, :call
  def call(env)
    machine = env[:machine]
    driver = machine.provider.driver
    uuid = driver.instance_eval { @uuid }
    ui = env[:ui]

    # Find out folder of VM
    vm_folder = ""
    vm_info = driver.execute("showvminfo", uuid, "--machinereadable")
    lines = vm_info.split("\n")
    lines.each do |line|
      if line.start_with?("CfgFile")
        vm_folder = line.split("=")[1].gsub('"','')
        vm_folder = File.expand_path("..", vm_folder)
        ui.info "VM Folder is: #{vm_folder}"
      end
    end

    disk_size = get_environment_variable('disk_size') || '20480'

    # add 2 to idx because sda is the boot disk and sdb is the cloud-init config drive
    # (without which boot will stall for up to 5 minutes)
    ('c'..'f').each_with_index do |disk, idx|
      disk_name = "sd#{disk}"
      disk_file = File.join(vm_folder, "#{disk_name}.vmdk")

      ui.info "Adding disk #{disk_name} to VM"
      if File.exist?(disk_file)
        ui.info "Disk already exists"
      else
        driver.execute("createmedium", "disk", "--filename", disk_file, "--size", disk_size, "--format", "vmdk")
        driver.execute('storageattach', uuid, '--storagectl', 'SCSI', '--port', (idx+2).to_s, '--type', 'hdd', '--medium', disk_file)
      end
    end

    original_call(env)
  end
end

kubernetes_version = get_environment_variable('kubernetes_version')

$generic_install_script = <<-EOH
  #!/bin/sh

  # Update sources.list
  if [ -f /vagrant/custom/sources.list ]; then
    cp /vagrant/custom/sources.list /etc/apt/sources.list
  fi

  # Update sources.list for kubernetes
  if [ -f /vagrant/custom/sources.list.kubernetes ]; then
    cp /vagrant/custom/sources.list.kubernetes /etc/apt/sources.list.d/kubernetes.list
    KUBEURL=$(cat /vagrant/custom/sources.list.kubernetes | grep '^deb\s' | awk '{ print $2 }' | head -n 1 | sed -e 's/"//g')
    curl -s "${KUBEURL}/doc/apt-key.gpg" | apt-key add -
  else
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list
  fi

  # Install Docker/Kubernetes
  apt-get update
EOH

$generic_install_script << \
  if kubernetes_version
    <<-EOH
  apt-get install -y \
  docker.io \
  kubelet=#{kubernetes_version}-00 \
  kubeadm=#{kubernetes_version}-00 \
  kubectl=#{kubernetes_version}-00 \
  kubernetes-cni
    EOH
  else
    <<-EOH
  apt-get install -y docker.io kubelet kubeadm kubectl kubernetes-cni
    EOH
  end

$generic_install_script << <<-EOH
  # Override repo prefix for kubelet if needed
  if [[ "$KUBE_REPO_PREFIX" != "" ]]; then
    OVERRIDE=/etc/systemd/system/kubelet.service.d/override.conf
    echo "[Service]" > $OVERRIDE
    echo "Environment='KUBELET_EXTRA_ARGS=--pod-infra-container-image ${KUBE_REPO_PREFIX}/pause-amd64:3.0'" >> $OVERRIDE

    systemctl daemon-reload && sleep 5 && systemctl restart kubelet
  fi
EOH

Vagrant.configure("2") do |config|
  kubeadm_token = "#{SecureRandom.hex[0...6]}.#{SecureRandom.hex[0...16]}"
  master_vm = 'master'
  master_vm_ip = '192.168.77.2'
  pod_network = '10.244.0.0/16'
  repo_prefix = get_environment_variable('repo_prefix')

  # retain base repo prefix for pulling flannel containers
  kubeadm_env = \
    if repo_prefix
      {
        "REPO_PREFIX" => repo_prefix,
        "KUBE_REPO_PREFIX" => repo_prefix + "/gcr.io/google_containers"
      }
    else
      {}
    end

  update_ca_certificates_script = ''

  http_proxy = get_environment_variable('http_proxy')
  https_proxy = get_environment_variable('https_proxy')
  no_proxy = get_environment_variable('no_proxy')

  if Vagrant.has_plugin?("vagrant-proxyconf") && (http_proxy || https_proxy || no_proxy)
    # copy proxy settings from the Vagrant environment into the VMs
    (no_proxy = [no_proxy, master_vm_ip].join(',')) unless no_proxy.nil?

    # vagrant-proxyconf will not configure anything if everything is nil
    config.proxy.http = http_proxy
    config.proxy.https = https_proxy
    config.proxy.no_proxy = no_proxy

    # look for another envvar that points to a directory with CA certs
    env_cacerts_dir = get_environment_variable('cacerts_dir')
    if env_cacerts_dir
      cacerts_dir = Dir.new(env_cacerts_dir)
      files_in_cacerts_dir = cacerts_dir.entries.select{ |e| not ['.', '..'].include? e }
      files_in_cacerts_dir.each do |f|
        next if File.directory?(File.join(cacerts_dir, f))
        begin
          unless f.end_with? '.crt'
            fail "All files in #{env_cacerts_dir} must end in .crt due to update-ca-certificates restrictions."
          end
          # read in the certificate and normalize DOS line endings to UNIX
          cert_raw = File.read(File.join(env_cacerts_dir, f)).gsub(/\r\n/, "\n")
          if cert_raw.scan('-----BEGIN CERTIFICATE-----').length > 1
            fail "Multiple certificates detected in #{File.join(env_cacerts_dir, f)}, please split them into separate certificates."
          end
          cert = OpenSSL::X509::Certificate.new(cert_raw) # test that the cert is valid
          dest_cert_path = File.join('/usr/local/share/ca-certificates', f)
          update_ca_certificates_script << <<-EOH
            echo -ne "#{cert_raw}" > #{dest_cert_path}
          EOH
        rescue OpenSSL::X509::CertificateError
          fail "Certificate #{File.join(env_cacerts_dir, f)} is not a valid PEM certificate, aborting."
        end # begin/rescue
      end # files_in_cacerts_dir.each
      update_ca_certificates_script << <<-EOH
        update-ca-certificates
      EOH
    end # if env_cacerts_dir
  end # if Vagrant.has_plugin?

  config.vm.box = "ubuntu/xenial64"
  config.vm.box_check_update = false

  config.vm.provision 'configure-extra-certificates', type: 'shell' do |s|
    s.inline = update_ca_certificates_script
  end unless update_ca_certificates_script.empty?

  config.vm.provision 'configure-swap-space', type: 'shell' do |s|
    s.inline = <<-EOH
      #!/bin/bash
      swap_path='/swap'
      swap_size="#{ENV['vm_swap_size'] ||= '2048'}M"
      # This could be a re-provision
      if grep -qw "^${swap_path}" /proc/swaps ; then
        swapoff "$swap_path"
      fi
      fallocate -l $swap_size "$swap_path"
      truncate -s $swap_size "$swap_path"
      chmod 600 "$swap_path"
      mkswap -f "$swap_path"
      /bin/sync
      swapon "$swap_path"
      if ! grep -qw "^${swap_path}" /etc/fstab ; then
        echo "$swap_path none swap defaults 0 0" | tee -a /etc/fstab
      fi
    EOH
  end

  config.vm.provision 'generic-install', type: 'shell' do |s|
    s.env = kubeadm_env
    s.inline = $generic_install_script
  end

  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true

  config.vm.define master_vm do |c|
    c.vm.hostname = master_vm
    c.vm.network 'private_network', ip: master_vm_ip

    cpu_count = 1
    env_cpu_count = ENV['master_cpu_count'].to_i
    (cpu_count = env_cpu_count) unless env_cpu_count.zero?

    memory_mb = 1024
    env_memory_mb = ENV['master_memory_mb'].to_i
    (memory_mb = env_memory_mb) unless env_memory_mb.zero?

    c.vm.provider "virtualbox" do |vb|
      vb.cpus = cpu_count
      vb.memory = memory_mb

      apply_vm_hardware_customizations(vb)
    end

    c.vm.provision 'kubeadm-init', type: 'shell' do |s|
      cmd = "kubeadm init --apiserver-advertise-address #{master_vm_ip} --pod-network-cidr #{pod_network} --token #{kubeadm_token}"
      s.env = kubeadm_env
      if kubernetes_version
        cmd += " --kubernetes-version=v#{kubernetes_version}"
      end
      s.inline = cmd
    end

    # use sudo here so that the id subshells get the non-root user
    c.vm.provision 'copy-kubeadm-config', type: 'shell' do |s|
      s.privileged = false
      s.inline = <<-EOH
        #!/bin/sh
        mkdir -p $HOME/.kube
        sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo cp -f /etc/kubernetes/admin.conf /vagrant/kube.config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
      EOH
    end

    env_network_provider = get_environment_variable('network_provider')
    network_provider = env_network_provider.nil? ? 'flannel' : env_network_provider.downcase

    fail "#{network_provider} is not a supported network provider" unless %w(flannel calico none).include?(network_provider)

    if network_provider == 'flannel'
      c.vm.provision 'install-flannel', type: 'shell' do |s|
        s.env = kubeadm_env
        s.privileged = false
        s.inline = <<-EOH
          #!/bin/sh

          if [ -f /vagrant/custom/kube-flannel-rbac.yml ]; then
            cp /vagrant/custom/kube-flannel-rbac.yml $HOME/kube-flannel-rbac.yml
          else
            curl -s -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
          fi

          if [ -f /vagrant/custom/kube-flannel.yml ]; then
            cp /vagrant/custom/kube-flannel.yml $HOME/kube-flannel.yml
          else
            curl -s -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
          fi

          export FLANNEL_IFACE=$(ip a | grep #{master_vm_ip} | awk '{ print $NF }')

          # substitute in the interface on our VM as the Flannel interface
          sed -r -i -e "s|command: \\[ \\"/opt/bin/flanneld\\", \\"--ip-masq\\", \\"--kube-subnet-mgr\\" \\]|command: \\[ \\"/opt/bin/flanneld\\", \\"--ip-masq\\", \\"--kube-subnet-mgr\\", \\"--iface\\", \\"${FLANNEL_IFACE}\\" \\]|" $HOME/kube-flannel.yml

          if [[ "$KUBE_REPO_PREFIX" != "" ]]; then
            sed -i -e "s|quay.io/coreos/flannel:|${REPO_PREFIX}/quay.io/coreos/flannel:|g" $HOME/kube-flannel.yml
          fi

          kubectl create -f $HOME/kube-flannel-rbac.yml
          sleep 2
          kubectl create -f $HOME/kube-flannel.yml
        EOH
      end # c.vm.provision 'install-flannel'
    elsif network_provider == 'calico'
      c.vm.provision 'install-calico', type: 'shell' do |s|
        s.env = kubeadm_env
        s.privileged = false
        s.inline = <<-EOH
          #!/bin/sh

          if [ -f /vagrant/custom/calico.yml ]; then
            cp /vagrant/custom/calico.yml $HOME/calico.yaml
          else
            curl -s -O http://docs.projectcalico.org/v2.3/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
          fi

          export CALICO_NETWORK=#{pod_network}

          # use the same network that we're specifying for Flannel installs (for consistency)
          sed -i -e "/CALICO_IPV4POOL_CIDR/{n;s|.*|              value: \\"$CALICO_NETWORK\\"|}" $HOME/calico.yaml

          # disable IP-in-IP
          sed -i -e '/CALICO_IPV4POOL_IPIP/{n;s/.*/              value: "off"/}' $HOME/calico.yaml

          kubectl create -f $HOME/calico.yaml
        EOH
      end # c.vm.provision 'install-calico'
    end # if network_provider
  end # config.vm.define master_vm

  worker_count = 3
  env_worker_count = ENV['worker_count'].to_i
  (worker_count = env_worker_count) unless env_worker_count.zero?

  worker_count.times do |i|
    idx = i+1

    vm_name = "w#{idx}"
    vm_ip = (master_vm_ip.split('.')[0..2] + [10+idx]).join('.')

    cpu_count = 1
    env_cpu_count = ENV['worker_cpu_count'].to_i
    (cpu_count = env_cpu_count) unless env_cpu_count.zero?

    memory_mb = 1024
    env_memory_mb = ENV['worker_memory_mb'].to_i
    (memory_mb = env_memory_mb) unless env_memory_mb.zero?

    config.vm.define vm_name do |c|
      c.vm.hostname = vm_name
      c.vm.network "private_network", ip: vm_ip

      c.vm.provider "virtualbox" do |vb|
        vb.cpus = cpu_count
        vb.memory = memory_mb

        apply_vm_hardware_customizations(vb)
      end

      c.vm.provision 'join-kubernetes-cluster', type: 'shell' do |s|
        kubeadm_join_cmd = "sudo kubeadm join --token #{kubeadm_token}"

        # if this envvar is defined in any way, skip kubeadm preflight checks
        kubeadm_join_cmd += ' --skip-preflight-checks' if get_environment_variable('skip_preflight_checks')
        kubeadm_join_cmd += " #{master_vm_ip}:6443"

        s.env = kubeadm_env
        s.inline = kubeadm_join_cmd
      end # c.vm.provision 'join-kubernetes-cluster'
    end # config.vm.define
  end # worker_count.times
end # Vagrant.configure
