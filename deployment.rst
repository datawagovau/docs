**********
Deployment
**********

This chapter explains the deployment of a Virtual Machine on the Amazon web services (AWS) cloud and hosting on a custom domain.
The goal is a VM with ssh access which runs a default web page on a custom domain, and provide a scalable way to add additional services.
We will build: 

* AWS EC2 t2.medium VM with at least 16 GB, if possible 100 GB hdd (the default of 8 GB is too small for OS and docker images)
* AWS 100 GB SSD volume, formatted with btrfs, mounted in above instance at e.g. ``/mnt/btrfsvol/`` with symlinks to locations containing large files
All non-ephemeral data should live inside ``/mnt/btrfsvol/``, which includes:
* the datacats installation
* the datacats data directory (soft-linked from default ~/.datacats)
* the docker image dir (soft-linked from default /var/lib/docker)
* Latex docs (> 1 GB)

The benefits:
* the external volume can be snapshotted for backup&recovery
* the external volume is btrfs-formatted, docker supports btrfs natively


AWS EC2 instance
================
In your AWS console, create a VM and a storage volume, attach the volume to the VM, create a static IP (ElasticIP).
You will want to have an SSH keypair configured at AWS.

* Login to AWS management console, e.g. `Sydney`_
* EC2 Dashboard
* Launch instance
* Ubuntu Server 14.04 LTS 64 bit (HVM), SSD Volume Type
* t2.medium
* Configure instance details
* Set shutdown behaviour to "stop", protect against accidental termination
* Set size of instance to 16 GB
* Add Storage: 100 GB SSD
* Tag instance (choose a name)
* Configure Security Group: SSH, HTTP, HTTPS (or select from existing security groups providing these three protocols)
* Review and Launch
* Select existing SSH key pair (or create new one)
* View instances

.. _Sydney: https://ap-southeast-2.console.aws.amazon.com/

Test: **AWS console > EC2 > Instances** should list new instance

Format the external volume::

  sudo su
  apt-get -y install btrfs-tools libxml2-utils gdal-bin
  
  fdisk -l
  # will show external, as yet unformatted volume as e.g. "xvdb" as "no partition table found"
  # your volume label will vary
  mkfs.btrfs /dev/xvdb
  mkdir -p /mnt/btrfsvol
  echo "/dev/xvdb /mnt/btrfsvol  btrfs rw,relatime,ssd,space_cache  0 0" >> /etc/fstab
  mount -a


Inspect new partition::

  df -h
  Filesystem      Size  Used Avail Use% Mounted on
  /dev/xvda1       16G  1.1G   14G   8% /
  none            4.0K     0  4.0K   0% /sys/fs/cgroup
  udev            2.0G   12K  2.0G   1% /dev
  tmpfs           396M  332K  395M   1% /run
  none            5.0M     0  5.0M   0% /run/lock
  none            2.0G     0  2.0G   0% /run/shm
  none            100M     0  100M   0% /run/user
  /dev/xvdb       100G  512K   98G   1% /mnt/btrfsvol


Folder structure
================
Locations of large files should be symlinked to the external volume ``/mnt/btrfsvol/``.
We will replace the following default locations with symlinks into the external volume:

* Apache's web app directory ``/var/www``
* Docker's image directory ``/var/lib/docker``
* A general purpose directory for projects, such as datacats
* Virtualenvwrapper's virtualenv directory (which we will configure in .bashrc) ``/var/venvs``
* Datacats' data directory ``~./datacats``
* User docs, including the sizeable Latex docs, will be moved from their original location ``/usr/share/doc`` to ``/var/www/doc`` and back-linked
* User ubuntu's home directory (accessible through RStudio Server) contains a symlink to ``/mnt/btrfsvol/shiny-server``

Create custom folders::

  sudo su
  
  mkdir -p /mnt/btrfsvol/www /mnt/btrfsvol/docker /mnt/btrfsvol/projects /mnt/btrfsvol/venvs /mnt/btrfsvol/datacats_data
  
  ln -s /mnt/btrfsvol/www /var/www
  ln -s /mnt/btrfsvol/docker /var/lib/docker
  ln -s /mnt/btrfsvol/projects /var/projects            
  ln -s /mnt/btrfsvol/venvs /var/venvs
  ln -s /mnt/btrfsvol/datacats_data /home/ubuntu/.datacats
  
  mv /usr/share/doc /var/www
  ln -s /var/www/doc /usr/share/doc
  
  chown -R ubuntu:www-data /mnt/btrfsvol /var/www /var/lib/docker /var/projects /var/venvs /home/ubuntu
  

Docker
======
Install docker into mounted btrfs volume, not the (16 GB) root volume, 
otherwise we'll run out of space (images for datacats + OS > 8 GB), and the docker install is not snapshotted.
In the previous step, we have create a folder inside the btrfsvol and symlinked docker's install default ``/var/lib/docker`` to that folder.
A symlink is preferred over a bind-mount via fstab, as docker will recognise and natively support btrfs through the symlink.
Following the prompt after the docker installation, add your non-root user to the group "docker" so they won't have to sudo docker commands.

Install docker::

  sudo su
  wget -qO- https://get.docker.com/ | sh
  usermod -aG docker ubuntu

You **need** to logout and login again to apply the group changes, else ``docker pull`` will fail, e.g. when pulling datacats images.

Web servers
===========
Install nginx, use nginx to reverse proxy subdomains to ports. Note that we won't need an Apache web server as all installed
packages come with their own web servers, but we'll include it to provide a hosting encironment for Apache apps.
If your custom setup requires Apache, make sure to change the default port in its ``/etc/apache2/ports.conf`` from 80 (which 
is used by our nginx) to some unused port, e.g. 8000.

Install nginx and apache::

  sudo apt-get install -y nginx apache2

Apache will be not used as web server unless your custom setup requires it; nginx will serve as reverse proxy for datacats, rstudio and rshiny.
Apache's document dir ``/var/www`` is a symlink pointing to the btrfs volume at ``/mnt/btrfsvol/www`` before apache2 is installed.
Apache's default listening port (80) will be changed to 8000 so nginx can listen on port 80.::

  sudo sed -i 's/Listen 80/Listen 8000/g' /etc/apache2/ports.conf
  sudo sed -i 's/<VirtualHost \*:80>/<VirtualHost \*:8000>/g' /etc/apache2/sites-enabled/000-default.conf

Add new nginx site `/etc/nginx/sites-enabled/base.conf`.
The DOMAIN.TLD is e.g. `yes-we-ckan.org` and we're running an Apache web server as well:

* requests to yes-we-ckan.org on port 80 will be redirected to port 8000 (apache site 1)
* other subdomains can be added similarly, redirecting to ports 8002 ff.

``/etc/nginx/sites-enabled/base.conf``::

  proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=cache:30m max_size=250m;
  proxy_temp_path /tmp/nginx_proxy 1 2;
  server {
      server_name DOMAIN.TLD www.DOMAIN.TLD;
      listen 80;
      location / {
          proxy_pass http://127.0.0.1:8000;
      }
  }
  server {
      server_name ckan.DOMAIN.TLD;
      listen 80;
      client_max_body_size 2G;
      location / {
          proxy_pass http://127.0.0.1:5000/;
          proxy_set_header X-Forwarded-For $remote_addr;
          proxy_set_header Host $host;
          proxy_cache cache;
          proxy_cache_bypass $cookie_auth_tkt;
          proxy_no_cache $cookie_auth_tkt;
          proxy_cache_valid 30m;
          proxy_cache_key $host$scheme$proxy_host$request_uri;
          # In emergency comment out line to force caching
          # proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
      }
  }
  
  server {
      server_name rstudio.DOMAIN.TLD;
      listen 80;
      location / {
          proxy_pass http://127.0.0.1:8787;
          proxy_redirect http://127.0.0.1:8787/ $scheme://$host/;
      }
  }
  server {
      server_name rshiny.DOMAIN.TLD;
      listen 80;
      location / {
          proxy_pass http://127.0.0.1:3838;
      }
  }
  
  server {
      server_name pycsw.DOMAIN.TLD;
      listen 80;
      location / {
          proxy_pass http://127.0.0.1:9000;
      }
  }
  
  server {
    server_name qgis.DOMAIN.TLD;
    listen 80;
    location / {
      proxy_pass http://127.0.0.1:8100;
      proxy_redirect http://127.0.0.1:8100/ $scheme://$host/;
    }
  }
  
  server {
    server_name openrefine.DOMAIN.TLD;
    listen 80;
    location / {
      proxy_pass http://127.0.0.1:3333;
      proxy_redirect http://127.0.0.1:3333/ $scheme://$host/;
    }
  }
  
  server {
    server_name ipython.DOMAIN.TLD;
    listen 80;
    location / {
      proxy_pass http://127.0.0.1:8888;
      proxy_redirect http://127.0.0.1:8888/ $scheme://$host/;
      proxy_set_header Origin http://127.0.0.1:8888;
    }
  }

Start apache2 and nginx services::
  service apache2 start
  service nginx start

Apache's default page should run on port 8000.
Run a test web server on port 8001 in the current directory with::

  cd /tmp
  python3 -m http.server 8001

``curl 127.0.0.1:8000`` should show the Apache default page (source)
``curl 127.0.0.1:8001`` should show a "directory listing" (source) if the python3 http.server is still running.
Mind that the AWS security group will only allow traffic on ports 22 (SSH), 80 (HTTP), and 443 (HTTPS).

Routing
=======
In Amazon's Route53, create a hosted zone, add aliases to your domain name.
In your domain registrar's management console, set your domain's name servers to those given to you by Route53 in your hosted zone.

* AWS management console > Route53 > Create Hosted Zone
* Name: your domain name, e.g. yes-we-ckan.org
* Add "A - IPv4 address" record with value (paste your Elastic IP) for both  `yes-we-ckan.org` and `*.yes-we-ckan.org`
* Make note of the NS (name servers)

Web domain
==========
Buy a domain, e.g. `yes-we-ckan.org`_ and allow a day or two to activate.
The domain registrar will allow to set custom name servers. 
Change the domain's DNS servers to your Route53 hosted zone's name servers (allow up to 48h to activate). E.g.:

  ns-1490.awsdns-58.org
  ns-521.awsdns-01.net
  ns-224.awsdns-28.com
  ns-1884.awsdns-43.co.uk


This will make requests to that domain use AWS's DNS servers, which will pick up Route 53's hosted zone pointing to your AWS VM. 
Nginx takes care of redirecting subdomains to internal ports.

* `yes-we-ckan.org`_ should show the Apache default page
* `open-data.yes-we-ckan.org`_ should show a "directory listing" if the python3 http.server is still running

.. _yes-we-ckan.org: http://yes-we-ckan.org
.. _open-data.yes-we-ckan.org: http://open-data.yes-we-ckan.org/

Virtualenv
==========
We'll install virtualenv and virtualenvwrapper as to keep installations of Python package requests 
contained in venv sandboxes, where they can't break pip.
We will also append custom virtualenvwrapper settings to ubuntu's bashrc.:

  sudo apt-get install -y python-pip
  sudo pip install virtualenvwrapper
  
  cat <<EOF >> /home/ubuntu/.bashrc
  export WORKON_HOME=/var/venvs
  export PROJECT_HOME=/var/projects
  source /usr/local/bin/virtualenvwrapper.sh
  EOF

Apply with logout/login or ``source ~/.bashrc``

Github SSH keys
===============
As user ubuntu, run ``ssh-keygen`` and register the public key from ``/home/ubuntu/.ssh/id_rsa.pub`` in your Github and / or 
Bitbucket accounts to enable ssh authentication. This will enable you to push to your code repositories.

To make connecting a local terminal to a remote VM using SSH easier, add to your local ssh config ``~/.ssh/config``::

  Host ALIAS
      HostName ec2-IP.NODE.compute.amazonaws.com
      User ubuntu
      IdentityFile ~/.ssh/MYKEY.pem


This allows to SSH into HostName as User with IdentityFile by simply typing ``ssh ALIAS``.
Capitalised terms will differ.

Disaster recovery
=================
On the AWS level, create snapshots of the VM and linked storage volumes. 
All servers should be shut down when the snapshots are taken.
We recommend maintaining a rolling daily snapshot for 30 days.

On the VM level, we recommend to:

* Export all data from CKAN, RStudio home directories, RShiny app directories to Amazon S3
* Script setup of server (= automate the steps of this documentation) and import of data from S3
* Schedule the backup daily or as required
* Create Amazon Auto Scaling Group, min and max instance no = 1 to create an "auto-rebuild" instance

If the instance should go down, the auto scaling group will create a new instance and restore the data from the S3 backup.

Result
======
Now we'll have an Amazon AWS VM to which we can 

* add servers running on custom ports (served through Apache2 or their own web servers), 
* reverse proxy the ports in Nginx to subdomain names like my-subdomain.yes-we-ckan.org,
* ``reload service nginx`` to apply these changes.
