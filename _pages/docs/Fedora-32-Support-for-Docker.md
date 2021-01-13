tl;dr Fedora 32 does not support Docker. Use a VM or moby.

# Background

Fedora 30 (2019 April) was the last release of Fedora where Docker installation was (supposedly) seamless.

Fedora 31 (2019 October) uses cgroup version 2 [1], which broke containers. There is a workaround [2]. Fedora 31 is the last release which is officially supported by Docker [3].

Fedora 32 (2020 April) **does not support Docker** [4].

[1] https://www.redhat.com/sysadmin/fedora-31-control-group-v2

[2] https://fedoraproject.org/wiki/Common_F31_bugs#Docker_package_no_longer_available_and_will_not_run_by_default_.28due_to_switch_to_cgroups_v2.29

[3] https://docs.docker.com/engine/install/fedora/

[4] https://fedoramagazine.org/docker-and-fedora-32/

Many of our collaborators use Fedora 32.

- https://github.com/FNNDSC/ChRIS_ultron_backEnd/issues/214 
- https://github.com/mairin/ChRIS_store/wiki/Getting-the-ChRIS-Store-to-work-on-Fedora

# Hands-off Vagrant Example

If this describes you:

- VirtualBox and Vagrant Installed
- 16 GB RAM
- [don't care](https://github.com/moby/moby/issues/22920#issuecomment-264036710), looking for code to copy-and-paste

Run docker in a VM, akin to how it works on Mac and Windows. I expect this demo to work on Ubuntu, Arch Linux, Mac, Windows, Fedora, RHEL

```bash
git clone https://github.com/FNNDSC/ChRIS_ultron_backEnd.git
cd ChRIS_ultron_backEnd
cat > Vagrantfile << EOF
Vagrant.configure("2") do |config|
  config.vm.box = "debian/buster64"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "8096"
  end
  config.vm.network "forwarded_port", guest: 8000, host: 8000 # cube
  config.vm.network "forwarded_port", guest: 8010, host: 8010 # store
  config.vm.network "forwarded_port", guest: 5005, host: 5005 # pfcon
  config.vm.network "forwarded_port", guest: 5010, host: 5010 # pman
  config.vm.network "forwarded_port", guest: 5055, host: 5055 # pfioh
  config.vm.provision "shell", inline: <<-SHELL
    wget -qO /usr/local/bin/docker-compose "https://github.com/docker/compose/releases/download/1.26.2/docker-compose-$(uname -s)-$(uname -m)"
    chmod +x /usr/local/bin/docker-compose
    wget -qO /tmp/get-docker.sh https://get.docker.com
    sh /tmp/get-docker.sh > /dev/null 2>&1
    systemctl enable --now docker
  SHELL
end
EOF
vagrant up
vagrant ssh -c 'cd /vagrant && sudo ./make.sh'
```

# SELinux

Relevant to Fedora, RHEL, CentOS.

SELinux enforcing and docker volumes will give you "permission denied" errors.
https://github.com/FNNDSC/ChRIS_ultron_backEnd/issues/214

According to `man docker-run`

> Labeling systems like SELinux require that proper labels are placed on volume content mounted into a container. Without a label, the security system might prevent the processes running inside the container from using the content. By default, Docker does not change the labels set by the OS.
> 
> To change a label in the container context, you can add either of two suffixes `:z` or `:Z` to the volume mount. These suffixes tell Docker to relabel file objects on the shared volumes. The z option tells Docker that two containers share the volume content. As a result, Docker labels the content with a shared content label. Shared volume labels allow all containers to read/write content.  The Z option tells Docker to label the content with a private unshared label.  Only the current container can use a private volume.

I created a branch [z-vol](https://github.com/FNNDSC/ChRIS_ultron_backEnd/compare/z-vol) to demonstrate how this works.

# Running `./make.sh` on docker-ce in Fedora 31

For the sake of showing how it is possible to run ChRIS on a Fedora machine with SELinux enabled, we will use a virtual machine. This section serves as just a proof, there's not much point in following these steps.

Use Vagrant to provision a lightweight **Fedora 31** VM and correctly install Docker.
https://docs.docker.com/engine/install/fedora/#install-docker-engine

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "fedora/31-cloud-base"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "8096"
  end

  config.vm.provision "shell", inline: <<-SHELL
    curl -L "https://github.com/docker/compose/releases/download/1.26.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose
    sh -c "$(curl -fsSL https://get.docker.com)"
    grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
    systemctl enable docker
    git clone --branch z-vol https://github.com/FNNDSC/ChRIS_ultron_backEnd.git
  SHELL
end
```

Create, provision, then reboot VM (required to set kernel cgroup option)

```
$ vagrant up && vagrant reload
```

Double check distro version and SELinux status

```bash
$ vagrant ssh -c 'cat /etc/fedora-release; getenforce'
Fedora release 31 (Thirty One)
Enforcing
Connection to 127.0.0.1 closed.
```

Start CUBE

```bash
$ vagrant ssh -c 'cd ChRIS_ultron_backEnd && sudo ./make.sh'
```

Result

```
Performing system checks...

System check identified no issues (3 silenced).
July 30, 2020 - 13:51:13
Django version 2.2.12, using settings 'config.settings.local'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.
```

# Running `./make.sh` on moby in Fedora 32

RedHat endorses [_podman_](https://podman.io/) to replace docker. However _podman_ does not support `docker-compose`. [podman-compose](https://github.com/containers/podman-compose) does not work.

For the insistent it is possible to use Fedora 32 and [moby](https://mobyproject.org/).

- https://fedoramagazine.org/docker-and-fedora-32/
- https://github.com/mairin/ChRIS_store/wiki/Getting-the-ChRIS-Store-to-work-on-Fedora

## Minimal Setup

```bash
sudo dnf install -y moby-engine docker-compose
sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
sudo systemctl enable docker
sudo systemctl reboot 
```

The steps are summarized here in this example using Vagrant again, for the sake of reproducibility.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "fedora/32-cloud-base"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "8096"
  end
  config.vm.provision "shell", inline: <<-SHELL
    dnf install -y moby-engine docker-compose git
    grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
    systemctl enable docker
    git clone --branch z-vol https://github.com/FNNDSC/ChRIS_ultron_backEnd.git
  SHELL
end
```

You need to start the machine and then reboot.

```bash
vagrant up && vagrant reload
```

Run `./make.sh`

```bash
vagrant ssh -c 'cd ChRIS_ultron_backEnd && sudo ./make.sh'
```

### Results

The last lines out output are

```
Performing system checks...

System check identified no issues (3 silenced).
July 31, 2020 - 12:24:26
Django version 2.2.12, using settings 'config.settings.local'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.
```

### Machine Information

```bash
$ vagrant ssh -c 'cat /etc/fedora-release && getenforce'
Fedora release 32 (Thirty Two)
Enforcing
```

# Comments

I tried Fedora 30 but `docker-compose` did not work out of the box.

Fedora 31 broke Docker, then Fedora 32 broke Docker even more. It has been 3 months since the release of Fedora 32 and it seems like Docker inc. is choosing not to support Fedora 32. Moby support for cgroup version 2 is in development: https://github.com/FNNDSC/ChRIS_ultron_backEnd/pull/213#issuecomment-666273966

[Fedora Silverblue](https://docs.fedoraproject.org/en-US/fedora-silverblue/toolbox/) sounds cool, check it out.

# Other Container Runtimes

Providing OCI configurations for compatibility with _podman_ and _kubernetes_ would be very nice.

