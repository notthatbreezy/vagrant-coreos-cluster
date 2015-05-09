# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.6"

Vagrant.configure("2") do |config|
  config.vm.box = "coreos-alpha"
  config.vm.box_url = "http://alpha.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json"

  config.vm.provider "virtualbox" do |v|
    v.check_guest_additions = false
    v.functional_vboxsf = false
  end

  (1..3).each do |index|
    config.vm.define "core-0#{index}" do |core|
      core.vm.hostname = "core-0#{index}"

      core.vm.provider "virtualbox" do |v|
        v.memory = 1024
        v.cpus = 1
      end

      core.vm.network :private_network, ip: "172.17.8.10#{index}"

      core.vm.provision "file", source: "cloud-config/core-0#{index}.yml", destination: "/tmp/vagrantfile-user-data"
      core.vm.provision "shell", inline: "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", privileged: true
    end
  end
end
