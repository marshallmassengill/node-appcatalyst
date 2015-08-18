# Set our default provider for this Vagrantfile to 'vmware_appcatalyst'
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'vmware_appcatalyst'

nodes = [
  { hostname: 'nodejs', box: 'vmware/photon' },

]

$ssl_script = <<SCRIPT


  echo Setting up SSL...

  mkdir -p /tmp/SSLCerts
  cd /tmp/SSLCerts
  openssl genrsa -aes256 -out ca-key.pem -passout pass:foobar 2048

  openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem -subj "/C=US/ST=GA/L=ATL/O=IT/CN=www.inkysea.com" -passin pass:foobar

  openssl genrsa -out server-key.pem 2048
  HOST=`hostname`
  openssl req -subj "/CN=$HOST" -new -key server-key.pem -out server.csr
  IP=`ifconfig eth0 | grep "inet\ addr" | cut -d: -f2 | cut -d" " -f1 `

  echo "subjectAltName = IP:$IP,IP:127.0.0.1" > extfile.cnf

  openssl x509 -req -days 365 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf -passin pass:foobar

  openssl genrsa -out key.pem 2048
  openssl req -subj '/CN=client' -new -key key.pem -out client.csr
  echo extendedKeyUsage = clientAuth > extfile.cnf
  openssl x509 -req -days 365 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile.cnf -passin pass:foobar

  rm -v client.csr server.csr

  mkdir -p /vagrant/DockerCerts/

  # Purge old certs on client
  chmod 755 /vagrant/DockerCerts/*
  rm /vagrant/DockerCerts/*

  # setup keys for IDE

  sudo -u vagrant cp -v {ca,cert,ca-key,key}.pem /vagrant/DockerCerts/
  chmod -v 0400 /vagrant/DockerCerts/*key.pem
  chmod -v 0444 /vagrant/DockerCerts/ca.pem
  chmod -v 0444 /vagrant/DockerCerts/cert.pem

  # Setup keys on docker host
  chmod -v 0400 ca-key.pem key.pem server-key.pem
  chmod -v 0444 ca.pem server-cert.pem cert.pem

  cp -v {ca,server-cert,server-key}.pem /etc/ssl/certs/
  mkdir -pv /root/.docker
  cp -v {ca,cert,key}.pem /root/.docker
  mkdir -pv /home/vagrant/.docker
  cp -v {ca,cert,key}.pem /home/vagrant/.docker
  echo "export DOCKER_HOST=tcp://$IP:2376 DOCKER_TLS_VERIFY=1" >> /etc/profile

  # Setup Docker.service and client
  SED_ORIG="ExecStart\\=\\/bin\\/docker \\-d \\-s overlay"
  SED_NEW="ExecStart\\=\\/bin\\/docker \\-d \\-s overlay \\-\\-tlsverify \\-\\-tlscacert\\=\\/etc\\/ssl\\/certs\\/ca\\.pem \\-\\-tlscert\\=\\/etc\\/ssl\\/certs\\/server\\-cert\\.pem \\-\\-tlskey\\=\\/etc\\/ssl\\/certs\\/server\\-key\\.pem \\--host 0\\.0\\.0\\.0\\:2376"
  sed -i "s/${SED_ORIG}/${SED_NEW}/" "/lib/systemd/system/docker.service"

  systemctl daemon-reload
  systemctl restart docker




SCRIPT


Vagrant.configure('2') do |config|

  # Configure our boxes with 1 CPU and 384MB of RAM
  config.vm.provider 'vmware_appcatalyst' do |v|
    v.vmx['numvcpus'] = '1'
    v.vmx['memsize'] = '512'
  end

  # Go through nodes and configure each of them.j
  nodes.each do |node|
    config.vm.define node[:hostname] do |node_config|
      node_config.vm.box = node[:box]
      node_config.vm.hostname = node[:hostname]
      node_config.vm.provision "shell", inline: $ssl_script
    end
  end
end
