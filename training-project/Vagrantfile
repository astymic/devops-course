# =============================================================================
# DevOps Course — Module 1: Dev Environment
# Vagrantfile — Infrastructure as Code for local VM
#
# Usage:
#   vagrant up      — create and start the VM
#   vagrant ssh     — connect to the VM
#   vagrant halt    — stop the VM
#   vagrant destroy — delete the VM completely
#
# Requirements: VirtualBox + Vagrant installed on the host machine
# =============================================================================

Vagrant.configure("2") do |config|

  # ── OS Image ──────────────────────────────────────────────────────────────
  # ubuntu/jammy64 = Ubuntu Server 22.04 LTS (Jammy Jellyfish)
  # LTS = Long Term Support — supported until April 2027; stable choice for
  # production-grade work, same as what we'd use on a real VPS.
  config.vm.box = "ubuntu/jammy64"

  # Human-readable hostname visible inside the VM (and in VirtualBox UI)
  config.vm.hostname = "devops-sandbox"

  # ── Network ───────────────────────────────────────────────────────────────
  # Private host-only network: VM is accessible from the host at 192.168.56.10
  # but NOT from the internet — safe for local experimentation.
  config.vm.network "private_network", ip: "192.168.56.10"

  # ── VirtualBox Provider Settings ──────────────────────────────────────────
  config.vm.provider "virtualbox" do |vb|
    vb.name   = "devops-sandbox"   # Name shown in VirtualBox Manager
    vb.memory = 2048               # RAM in MB (2 GB)
    vb.cpus   = 2                  # Number of virtual CPU cores

    # Display VM name in VirtualBox topbar
    vb.customize ["modifyvm", :id, "--groups", "/DevOps Course"]
  end

  # ── Initial Provisioning ─────────────────────────────────────────────────
  # This shell script runs ONCE on first `vagrant up` to prepare the VM.
  # This is a minimal bootstrap — full server setup will be done with Ansible
  # in Module 6 (Infrastructure as Code at scale).
  config.vm.provision "shell", inline: <<-SHELL
    echo "============================================="
    echo "  🚀 DevOps Sandbox — Initial Setup"
    echo "============================================="

    # Update package lists
    export DEBIAN_FRONTEND=noninteractive
    apt-get update -q

    # Install essential tools
    apt-get install -y -q \
      curl \
      git \
      htop \
      net-tools \
      vim

    echo ""
    echo "============================================="
    echo "  ✅ VM is ready! Useful commands:"
    echo "     vagrant ssh   — connect to this VM"
    echo "     vagrant halt  — stop the VM"
    echo "     uname -a      — check OS version"
    echo "============================================="
  SHELL

end
