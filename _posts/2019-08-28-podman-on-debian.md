---
layout: post
title: "A Docker Daemon?  On *My* 1 GB VM?"
date: 2019-08-28
---

I've made it a goal in my life to build the leanest possible cloud provider based internet service.  One that is capable of serving up my sites without raping me on bandwidth costs and storage fees.  I believe I've designed it somewhat correctly so I think I'll detail how it's possible piecemeal when I feel like it.  I'm not against owning my own machines but since I've only built one computer in my whole life and never had a career figuring solving hardware issues I'd rather not start now when there's a huge target of private media for everyone, everywhere, any time.  I don't need distractions while devouring my whale, so to speak.

The first and most basic part of this is optimizing for the size of the instances, or droplets or servers that I'd be using.  The $5 ones that DigitalOcean and Vultr seem to have the features that in combination with their participation in the Bandwidth Alliance will make this possible, Linode is another possibility.  There are also a few other providers outside of the BA that would work but I'm saving those for myself in case Cloudflare goes full corporate and kills my ability to use tons of bandwidth for minimal expense after they IPO.  The immediate problem is my resource budget - I can't fit most of the new fangled software people are creating without allocating space that I'd rather use on actually serving users.  Kubernetes?  They recommend 2GB _minimum_ for a worker node.  Docker?  Their daemon takes up 70 MB.  On a $5 1GB node of which I will be using many, getting an extra 7% of ram to utilize may matter...or it may not.  But I already did it so let's roll.

The star of this dockerless show is Podman. Some people may wonder why I'm not using buildah as well.  Well, I still like using docker on my laptop since I have 32GB of ram to be sloppy with and I don't like spending time on things I don't care about.  When docker makes a breaking change I'll swap.  Podman is a daemonless container running, pushing, pulling system (and much more) and handles that part of the container management process with ease.  I'm using Vultr, so here's a quick tutorial on getting up and running with containers on their service.  You can do roughly the same thing on any other provider.  These steps are for linux.  Sorry, everyone else.

Part 1
<ol>

<li> Run this script on your local machine.  It assumes ubuntu or debian.  If you're running something else them I assume you're advanced or dedicated enough to make it work regardless.  make sure you know what it's doing if you don't trust me.  You really shouldn't trust anyone.  If you need to make it executable just do chmod +x install_podman.sh.  This all might take a while.

install_podman.sh
<pre>
#!/bin/bash
sed -i 's/main/main contrib non-free/g' /etc/apt/sources.list
apt update && apt upgrade -y && \
DEBIAN_FRONTEND=noninteractive apt -qy install linux-headers-$(uname -r) \
  btrfs-tools \
  bridge-utils \
  systemd-container \
  gcc \
  make \
  cmake \
  git \
  btrfs-progs \
  golang-go \
  go-md2man \
  iptables \
  libassuan-dev \
  libc6-dev \
  libdevmapper-dev \
  libglib2.0-dev \
  libgpgme-dev \
  libgpg-error-dev \
  libostree-dev \
  libprotobuf-dev \
  libprotobuf-c-dev \
  libseccomp-dev \
  libselinux1-dev \
  libsystemd-dev \
  pkg-config \
  runc \
  uidmap \
  libapparmor-dev

if [ ! -d "$HOME" ]; then
  export HOME=/home/USERNAME
fi
if [ ! -d "$GOPATH" ]; then
  export GOPATH=$HOME
fi

# install conmon for podman
git clone https://gitlab.com/iffpodmanfreeze/conmon.git
cd conmon
make
make podman
cp /usr/local/libexec/podman/conmon  /usr/local/bin/

# install CNI plugins for podman
git clone https://gitlab.com/iffpodmanfreeze/containernetworkingplugins.git $GOPATH/src/github.com/containernetworking/plugins
cd $GOPATH/src/github.com/containernetworking/plugins
./build_linux.sh
mkdir -p /usr/libexec/cni
cp bin/* /usr/libexec/cni

# setup CNI networking
mkdir -p /etc/cni/net.d/
touch /etc/cni/net.d/99-loopback.conf
cat &lt;&lt;EOT &gt; /etc/cni/net.d/99-loopback.conf
{
  "cniVersion": "0.4.0",
  "name": "podman",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni-podman0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "routes": [
          {
            "dst": "0.0.0.0/0"
          }
        ],
        "ranges": [
          [
            {
              "subnet": "10.88.0.0/16",
              "gateway": "10.88.0.1"
            }
          ]
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    },
    {
      "type": "firewall",
      "backend": "iptables"
    }
  ]
}
EOT

#Populate registry configuration file
mkdir -p /etc/containers
touch /etc/containers/registries.conf
cat &lt;&lt;EOT &gt; /etc/containers/registries.conf
# This is a system-wide configuration file used to
# keep track of registries for various container backends.
# It adheres to TOML format and does not support recursive
# lists of registries.

# The default location for this configuration file is /etc/containers/registries.conf.

# The only valid categories are: 'registries.search', 'registries.insecure', 
# and 'registries.block'.

[registries.search]
registries = ['docker.io', 'registry.fedoraproject.org', 'registry.access.redhat.com', 'registry.gitlab.com']

# If you need to access insecure registries, add the registry's fully-qualified name.
# An insecure registry is one that does not have a valid SSL certificate or only does HTTP.
[registries.insecure]
registries = []

# If you need to block pull access from a registry, uncomment the section below
# and add the registries fully-qualified name.
#
# Docker only
[registries.block]
registries = []
EOT

#Populate registry policy config
mkdir -p /etc/containers
touch /etc/containers/policy.json
cat &lt;&lt;EOT &gt; /etc/containers/policy.json
{
  "default": [
    {
      "type": "insecureAcceptAnything"
    }
  ],
  "transports": {
    "docker-daemon": {
      "": [
        {
          "type": "insecureAcceptAnything"
        }
      ]
    }
  }
}
EOT

#We can install Podman now!!!
git clone https://gitlab.com/iffpodmanfreeze/libpod.git $GOPATH/src/github.com/containers/libpod
cd $GOPATH/src/github.com/containers/libpod
make
sudo make install
cd ..
tar xcf ~/libpod.tar.gz ./libpod 
#debian thing for kernel namespaces
echo "kernel.unprivileged_userns_clone=1" >> /etc/sysctl.d/local.conf
echo "GOOD TO GO."
</pre>

It installed podman on your local machine and it also copied the tar'd libpod directory to your home directory.  Why?  Because we're sending it to the cloud.
</li>
<li>Make sure you have an account on Vultr or some other cloud provider with $5 instances.</li>

</ol>
Part 2

<ol>
<li>Make sure you have an SSH key on your local machine.</li>

<li>Go to your Vultr Account and create a startup script.  Paste the below code in it.  Call it 'setup' then save and go back the products page.</li>

<pre>
	#!/bin/bash
	adduser --disabled-password --gecos "" ssh_username;
	echo "ssh_username:higgybiggyboo" | chpasswd
	usermod -aG sudo ssh_username;
	sed -re 's/^(\#?)(PasswordAuthentication)([[:space:]]+)yes/\2\3no/'  -i.`date -I` /etc/ssh/sshd_config;
	sed -re 's/^(\#?)(PermitRootLogin)([[:space:]]+)yes/\2\3no/'  -i.`date -I` /etc/ssh/sshd_config;

	#dev sshkey setup
	sudo su - ssh_username;
	cd /home/ssh_username/ && mkdir .ssh && cd .ssh;
	echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC+aZrJIlQU0Tf7PVAnTAb58BjTHTyAXRWDfUGigOGSS5HIAVQBz/q4h2zaqUGr6v+FMEbOV05aTS0deta9kyruwphvCkux3cA9Nsa8NdjmS/ayzUm22hC+jeFsUOCmPMcgXSPm5iOqpOUiiRqfJe11h+0MuyMlovptlSbom2uCbD1MIlq1jDSZeQlpSY1GveiZ8lD1dKesOdYe7GcXY6fC2wost646lbSS8zS+IsaLLDt7xB7UvV3acsrjSBhuo02xufnPjkXxPW3rfXYToES51KdwEuVdBE8486FF4ZJLYbay4rnHRSu6mAELTY9NXSBEojl9pS9+MgPILk1OJ7D5ZCafJZ4PaMx0ZC3hnRFB2AoBZUBYOB1sFp6wxiyL+Bh2jDXZuM2MMZ4MWMHpQgwjTQEKFSCvVll4+rHjz2qAcg6bpFC8ju0zdnYzo05LS9cG5T9NrlCZZjC1hnVC17L9dheIZaFrIS6oOxCmEBWHguKvMubv5OvUh729gbLc/CX3NTGc9xUuJ+RUOW9SeTindboKcjXzWFxaD0aziqwwgyxbCmJ+bb3d5IPt0VnBFlH/lDWRe2ATbmwMqpiwvKKZMpscGq5SHzX1NPYad1qCm/5O5X+dshZEOMnnQLQcdtcBx3NYiJWIxoveCUFSYVke0NQyggn/wqqabWvBhMHIPQ== amazinguser@userslappy" > authorized_keys
	systemctl reload sshd.service

	echo "GOOD TO GO."
</pre>

That's the bare minimum for a somewhat secure ssh accessible box.  Change the password as well.  You could probably install ufw and lock it down to your ip but this is for learning.

<li>Back at the products page deploy a server anywhere you want.
	Choose the $5 1GB Debian 10 instance.  Thriftiness is a virtue.
	Select IPV6 because we live in the future.
	Select private networking just in case because it can be much faster with multiple services
	Select your 'setup' script.  You can also add your ssh key but we're doing it in the setup script because it's more portable.  It doesn't hurt though.
	Deploy Now.  You're now being charged.  Go faster.</li>

<li>Copy the ip address and SSH into the server when it's running, you'll only have to wait a few seconds.
<pre>
	ssh ssh_username@123.456.789.012
</pre>
</li>
<li>If the previous step worked Open up a new terminal and scp the 'install_podman.sh' script to the server with the code below.  If not, then figure out why it didn't work.
<pre>
	scp install_podman.sh ssh_username@123.456.789.012:~/
</pre>
</li>
<li>In your previous terminal become sudo with:
<pre>
	sudo su
</pre>
type in your password.
</li>
<li>Run the 'install_podman.sh' and wait for that bitch to Out of Memory Error.  The reason it OOMs is because the go build command is running out of memory like the gluttonous modern day software it is.  Seriously people, get your shit together - 1GB of memory should be enough for anyone to compile code.  We have SSDs, page that shit.  The final product works well so we're still going to use it.  We're not going to fix the OOM error though, that's too technical and we have other things to build.
</li>
<li>
scp that libpod.tar.gz in your home directory on your local machine to the server.  Caution: this step was only tested between ubuntu and debian. if you're running something else locally then figure out how to cross compile go code or make it in a container or another server or something.
<pre>
	scp libpod.tar.gz ssh_username@123.456.789.012:~/
</pre>
</li>

<li>untar the file, cd into the directory and modify the Makefile to just copy the completed files to the server.
<pre>
	tar xzf libpod.tar.gz && cd libpod
	vim Makefile. # or nano or whatever
</pre>
Just find these lines:
<pre>
install.remote: podman-remote
install.bin: podman
install.man: docs
install.config:
install.completions:
install.cni:
install.docker: docker-docs
install.systemd:
</pre>
delete everything to the right of the colon on those lines, i.e. podman-remote, podman, docs, and docker-docs should be removed.  Save and exit.
</li>
<li>
run this, you should still be sudo, if you aren't then prepend sudo the command:
<pre>
	make install
</pre>
</li>
<li>
reboot the machine
</li>
<li>
ssh to the machine again and type
<pre>
	podman pull alpine
</pre>
</li>
</ol>
If it worked, then congrats you now have a docker replacement on your machine and more ram to use in the future.  You should delete everything in your home directory and snapshot the machine so you never have to do this again.  If you have a more powerful machine then you won't oom and there's no need for these extra copy steps.  Go <a href="https://podman.io/blogs/">here</a> to learn more or type 'man podman' in your terminal.  It's a well built piece of software.

Next time I'll probably write something about flexible low resource high performance servers.  Nginx, h2o that sort of thing.