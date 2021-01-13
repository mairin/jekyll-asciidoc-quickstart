
Tested on **OSX 10.11.6** with **Vagrant 1.8.4**

**Vagrant >= 1.8.4 is REQUIRED [(direct link)](https://www.vagrantup.com/downloads.html)**

1- Get chris-ultron-backend (optional)
```
<host>$ mkdir -p ~/work/gitroot
<host>$ cd ~/work/gitroot
<host>$ git clone https://github.com/FNNDSC/ChRIS_ultron_backEnd.git chris-ultron-backend
```

2- Create the vagrant working directory
```
<host>$ mkdir -p ~/work/vagrant/chris-ultron
```

3- Get the configuration scripts from the `chris-ultron-backend` source.

**Adjust configuration to your setup in the Vagrantfile**
```
<host>$ cd ~/work/vagrant/chris-ultron
<host>$ cp -rv ~/work/gitroot/chris-ultron-backend/utils/vagrant/* .
```

4- Install the vagrant box
```
<host>$ vagrant box add ubuntu/xenial64
```

5- Install Virtual box guest additions to share folders
```
<host>$ vagrant plugin install vagrant-vbguest
```

6- Start vagrant
```
<host>$ vagrant up
```

7- Go grab a cup of coffee
```
It takes a while to install PIP...
```

8- Setup the Django DB
```
<host>$ vagrant ssh
<guest>(chris-ultron)$ cd ~/chris-ultron-backend/chris_backend
<guest>(chris-ultron)$ python manage.py migrate
```

9- Test the setup
```
<guest>(chris-ultron)$ python manage.py test
```

10- Create a chris user and test user
```
<guest>(chris-ultron)$ python manage.py createsuperuser --username chris --email chris@chris.ultron
password: Chris1234
<guest>(chris-ultron)$ python manage.py createsuperuser --username ubuntu --email user@chris.ultron
password: chrisultron
```

11- Start the Django server
```
<guest>(chris-ultron)$ python manage.py migrate
<guest>(chris-ultron)$ python manage.py runserver 0.0.0.0:8000
```

12- Create a Plugin
```
<host>$ brew install httpie
<host>$ http -a chris:Chris1234 POST http://127.0.0.1:8001/api/v1/plugins/ Content-Type:application/vnd.collection+json Accept:application/vnd.collection+json template:='{"data":[{"name":"name","value":"Feed1"}, {"name":"plugin","value":1}, {"name":"type","value":"fs"}]}'
```

13- Retrieve the Feed collection.
```
<host>$ http -a ubuntu:chrisultron  http://127.0.0.1:8001/api/v1/plugins
```

14- (Optional) Extra configuration for NFS setup
* Edit the Vagrant file to uncomment the relevant lines
* Add vagrant sudo commands
```
<host>$ sudo visudo
  Cmnd_Alias VAGRANT_EXPORTS_ADD = /usr/bin/tee -a /etc/exports
  Cmnd_Alias VAGRANT_NFSD = /sbin/nfsd restart
  Cmnd_Alias VAGRANT_EXPORTS_REMOVE = /usr/bin/sed -E -e /*/ d -ibak /etc/exports
  %admin ALL=(root) NOPASSWD: VAGRANT_EXPORTS_ADD, VAGRANT_NFSD, VAGRANT_EXPORTS_REMOVE
```
* Map chris-ultron IP to hostname (for convenience)
```
<host>$ vim /private/etc/hosts
  192.168.33.10   chris-ultron
```