# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'securerandom'

@net_prefix = '10.79.29'
@cluster_size = 3
@install_script = <<-EOF
  if [[ ! -f /usr/bin/kubeadm ]]; then
    apt-get clean
    rm -rf /var/lib/apt/lists/*
    apt-get update && apt-get full-upgrade
    apt-get install -y apt-transport-https
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list
    apt-get update
    apt-get install -y docker-engine
    apt-get install -y kubelet kubeadm kubectl kubernetes-cni
  fi
EOF

# Get the token needed to add nodes to the cluster, generating a new one if
# necessary.
def token
  unless File.exist?('.token')
    token = SecureRandom.hex(3) + '.' + SecureRandom.hex(8)
    File.open('.token', File::WRONLY | File::TRUNC | File::CREAT, 0o0600) do |f|
      f.write(token)
    end
  end
  File.open('.token').read
end

# Create new cluster node.
#   node_number: arbitrary number used to name nodes and give them IP addresses
#   config: a Vagrant config object
#   shell_script: an optional shell script with which to provision this node (in
#                 addition to the basic "install script" for Kubernetes).
def create_node(node_number, config, shell_script = nil)
  ip_address = "#{@net_prefix}.#{100 + node_number}"
  config.vm.define "node-#{node_number}" do |node|
    node.vm.hostname = "node-#{node_number}"
    node.vm.network 'private_network', ip: ip_address
    node.vm.provision :shell, inline: shell_script if shell_script
  end
  { ip: ip_address }
end

Vagrant.configure('2') do |config|
  config.cache.scope = :box if Vagrant.has_plugin?('vagrant-cachier')
  config.vm.box = 'bento/ubuntu-16.04'
  config.vm.provider 'virtualbox' do |vb|
    vb.memory = '4096'
  end
  config.vm.provision 'shell', inline: @install_script

  # Initialize the master node.
  master = create_node(
    1, config,
    "test -f /etc/kubernetes/admin.conf || kubeadm init --token=#{token} " \
    "--apiserver-advertise-address=`ip addr | egrep --only-matching '#{@net_prefix}[.][0-9]+'` &&" \
    "grep -q '^KUBECONFIG' /etc/environment || echo 'KUBECONFIG=/etc/kubernetes/admin.conf' > /etc/environment"
  )

  # Build worker nodes and introduce them to the cluster.
  (2..@cluster_size).each do |n|
    create_node(n, config, "kubeadm join --token=#{token} #{master[:ip]}:6443")
  end
end
