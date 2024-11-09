# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define :servidorWeb do |servidorWeb|
    servidorWeb.vm.box = "bento/ubuntu-22.04"
    servidorWeb.vm.network :private_network, ip: "192.168.60.3"
    servidorWeb.vm.provision "file", source: "webapp", destination: "/home/vagrant/webapp"
    servidorWeb.vm.provision "file", source: "init.sql", destination: "/home/vagrant/init.sql"
    servidorWeb.vm.provision "shell", path: "script.sh"
    servidorWeb.vm.hostname = "servidorWeb"
  end
  config.vm.define :prometheus do |prometheus|
    prometheus.vm.box = "bento/ubuntu-22.04"
    prometheus.vm.network :private_network, ip: "192.168.60.4"
    prometheus.vm.hostname = "prometheus"
  end
end
