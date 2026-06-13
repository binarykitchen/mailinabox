# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "cloud-image/ubuntu-26.04"

  # libvirt/QEMU provider support. The cloud-image/ubuntu-26.04 box ships both
  # a VirtualBox and a libvirt image, so no box override is needed.
  config.vm.provider :libvirt do |libvirt, override|
    # The box's bundled Vagrantfile disables the default /vagrant share, so
    # re-enable it. Use rsync, not 9p: setup/munin.sh chmods a symlink that
    # resolves into the shared folder, which 9p rejects (chmod returns EPERM
    # even on a 26.04 guest, because the host-side export is not owner-mapped).
    # rsync copies the tree onto the guest's own filesystem where chmod just
    # works. Trade-off: one-way host->guest sync (`vagrant rsync` to push later
    # edits), which is fine for a provisioning test box.
    override.vm.synced_folder ".", "/vagrant", type: "rsync", disabled: false
  end

  # Network config: Since it's a mail server, the machine must be connected
  # to the public web. However, we currently don't want to expose SSH since
  # the machine's box will let anyone log into it. So instead we'll put the
  # machine on a private network.
  config.vm.hostname = "mailinabox.lan"
  config.vm.network "private_network", ip: "192.168.56.4"

  config.vm.provision :shell, :inline => <<-SH
    # Set environment variables so that the setup script does
    # not ask any questions during provisioning. We'll let the
    # machine figure out its own public IP.
    export NONINTERACTIVE=1
    export PUBLIC_IP=auto
    export PUBLIC_IPV6=auto
    export PRIMARY_HOSTNAME=auto
    #export SKIP_NETWORK_CHECKS=1

    # Start the setup script.
    cd /vagrant
    setup/start.sh

    # MiaB's `ufw limit ssh` REJECTs SSH after 6 connections/30s from one
    # source (recent --update --hitcount 6, reject-with port-unreachable).
    # On a reboot of an already-set-up box, Vagrant's SSH-readiness check
    # polls 192.168.121.x:22 faster than that, trips the limit, and the
    # --update keeps the window alive so it never recovers: Vagrant hits
    # boot_timeout and force-halts the VM. Exempt the libvirt management
    # subnet from the rate-limit. No-op on the VirtualBox box, which reaches
    # the guest over NAT and has no 192.168.121.x interface.
    if ip -o -4 addr show | grep -q '192.168.121.'; then
        ufw status | grep -q '192.168.121.0/24' || ufw insert 1 allow proto tcp from 192.168.121.0/24 to any port 22
    fi
SH
end
