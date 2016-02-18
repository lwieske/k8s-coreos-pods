# -*- mode: ruby -*-
# vi: set ft=ruby :

members = {
  #  Name     #, CPU,  RAM, 1ST_IP
  'etcd' => [ 4,   1,  256,     11 ],
  'node' => [ 4,   2, 2048,    100 ],
}
PREFIX   = "10.10.10"
IP_RANGE = "10.11.0.0/16"

Vagrant.configure("2") do |config|
  config.vm.box     = "coreos-alpha"
  config.vm.box_url = "http://alpha.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json"

  INITIAL_CLUSTER = Array.new(members['etcd'][0]) { |i| "%s-%02d=http://%s.%02d:2380" % ['etcd', i+1, PREFIX, members['etcd'][3]+i] } * ","
  ETCD_SERVERS    = Array.new(members['etcd'][0]) { |i| "http://%s.%02d:2379"         % [PREFIX, members['etcd'][3]+i] }              * ","
  KUBE_MASTER     = PREFIX + "." + members['node'][3].to_s

  members.each do |name, (count, cpu, ram, ipstart)|
    (1..count).each do |i|
      config.vm.define member = "%s-%02d" % [name, i] do |m|
        m.vm.hostname = member
        config.vm.provider :virtualbox do |v|
          v.name                  = member
          v.cpus                  = cpu
          v.memory                = ram
          v.check_guest_additions = false
          v.functional_vboxsf     = false
        end
        ip = "%s.%02d" % [PREFIX, ipstart+i-1]
        m.vm.network :private_network, ip: ip
        role = i == 1 ? "master" : "minion"

        m.vm.provision :file, :source => "./#{name}.yaml", :destination => "/tmp/cloud-config.yaml"
        m.vm.provision :shell, inline: <<-SCRIPT
              sed -i -e "s|__HOSTNAME__|#{member}|" \
                     -e "s|__HOST_IP__|#{ip}|" \
                     -e "s|__INITIAL_CLUSTER__|#{INITIAL_CLUSTER}|" \
                     -e "s|__ETCD_SERVERS__|#{ETCD_SERVERS}|" \
                     -e "s|__IP_RANGE__|#{IP_RANGE}|" \
                     -e "s|__METADATA__|core=node,kube=#{role}|" \
              /tmp/cloud-config.yaml
        SCRIPT

        if (name == "node") then
          metadata = i == 1 ? "master" : "minion"
          m.vm.synced_folder "./pods/#{role}", "/etc/kubernetes/manifests/", type: "rsync"
          m.vm.provision :shell, inline: <<-SCRIPT
              sed -i -e "s|__KUBE_MASTER__|#{KUBE_MASTER}|" \
                     -e "s|__IP_RANGE__|#{IP_RANGE}|" \
              /etc/kubernetes/manifests/kube.yaml
          SCRIPT
        end
        m.vm.provision :shell, :inline => "echo COREOS_ETCD_SERVERS=#{ETCD_SERVERS} >>/etc/environment", :privileged => true
        m.vm.provision :shell, :inline => "echo FLANNEL_IP_RANGE=#{IP_RANGE}        >>/etc/environment", :privileged => true
        m.vm.provision :shell, :inline => "echo KUBE_MASTER=#{KUBE_MASTER}          >>/etc/environment", :privileged => true

        m.vm.provision :shell, :inline => "mv /tmp/cloud-config.yaml /var/lib/coreos-vagrant/vagrantfile-user-data", :privileged => true
      end
    end
  end
end
