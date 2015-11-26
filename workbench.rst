**********************
Data Science Workbench
**********************

RStudio Server
==============
Follow the rocker-rstudio wiki at https://github.com/rocker-org/rocker/wiki/Using-the-RStudio-image
to run the RStudio Server image::

  mkdir -p /var/projects/rstudio-server
  docker run -d \
  -p 8787:8787 \
  -e USER=<username> \
  -e PASSWORD=<password> \
  -v /var/projects/rstudio-server/:/home/ \
  rocker/ropensci


Login to [RStudio Server]() with above set username and password and install a few packages::

  install.packages(c('shinyIncubator','markdown','whisker','Hmisc','qcc','httr','RCurl','curl','devtools'), repos='http://cran.rstudio.com/')
  devtools::install_github("ropensci/ckanr")
  ckanr::ckanr_setup(url="http://catalogue-beta.data.wa.gov.au/")

Follow rocker-rstudio's instructions at https://github.com/rocker-org/rocker/wiki/Using-the-RStudio-image#multiple-users
to add new users for your (public-facing) RStudio Server.

RShiny Server
=============
Following the  rocker-rshiny wiki https://github.com/rocker-org/shiny::

  mkdir -p /var/projects/shiny-logs
  mkdir -p /var/projects/shiny-apps
  
  docker run -d \
  -p 3838:3838 \
  -v /var/projects/shiny-apps/:/srv/shiny-server/ \
  -v /var/projects/shiny-logs/:/var/log/ \
  -v /usr/local/lib/R/site-library/:/usr/local/lib/R/site-library/ \
  rocker/shiny

  cd /var/projects/shiny-apps
  git clone git@github.com:florianm/ckan-pip-collisions.git


* Add shiny apps into ``/var/projects/shiny-server``.
* Find logs from rocker-rshiny in ``/var/projects/shiny-logs``.
* Install R packages as required by your rshiny apps into ``/usr/local/lib/R/site-library``::

  sudo su
  apt-get -y install r-base r-base-dev gdebi-core texlive-full htop ncdu libcurl4-openssl-dev libxml2-dev

  sudo su - -c "R -e \"install.packages(c('bitops','caTools','colorspace','RColorBrewer','scales','ggplot2','rjson','markdown','knitr','stringr',
  'shiny','shinyIncubator','markdown','whisker','Hmisc','dplyr','tidyr','lubridate','qcc','httr','RCurl','curl','devtools'), 
  repos='http://cran.rstudio.com/');devtools::install_github('ropensci/ckanr')\""

  # or as root in an R session:
  shopping_list = c('bitops','caTools','colorspace','RColorBrewer','scales','ggplot2','rjson','markdown','knitr','stringr',
  'shiny','shinyIncubator','markdown','whisker','Hmisc','dplyr','tidyr','lubridate','qcc','httr','RCurl','curl','devtools')
  install.packages(shopping_list, repos='http://cran.rstudio.com/')
  devtools::install_github('ropensci/ckanr')


# CKAN-o-Sweave
Using Rstudio Server, follow https://github.com/datawagovau/ckan-o-sweave to fork yourself a nice cold CKAN-o-Sweave.
Read the instructions in the README to get started, and read the example report for an in-depth explanation.


Following the  RShiny Server docs at https://support.rstudio.com/hc/en-us/articles/200552326-Configuring-the-Server/,
add to ``/etc/nginx/sites-enabled/base.conf`` (substituting DOMAIN.TLD with your domain)::


  server {
    server_name rshiny.DOMAIN.TLD;
    listen 80;
    location / {
      proxy_pass http://127.0.0.1:3838;
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

OpenRefine
==========

Using https://github.com/SpazioDati/docker-openrefine 's container::

  mkdir -p /var/projects/openrefine
  docker run -d -p 3333:3333 -v /var/projects/openrefine/:/mnt/refine/ spaziodati/openrefine
  
  # with custom DNS server:
  #docker run -d -p 3333:3333 -v /var/projects/openrefine/:/mnt/refine/ --dns=10.6.20.100 spaziodati/openrefine



nginx conf::

  server {
    server_name openrefine.DOMAIN.TLD;
    listen 80;
    location / {
      proxy_pass http://127.0.0.1:3333;
      proxy_redirect http://127.0.0.1:3333/ $scheme://$host/;
    }
  }

Integration suggestion:

* CSV and XLS resources menu: "OpenRefine" link
* Use OpenRefine API to create new project at openrefine.beta.data.wa.gov.au using resource url

Or OpenRefine View: create OR projects for all data resources with ``paster views openrefine`` and persist OR urls in resource view

IPython Notebook Server
=======================
Using https://github.com/ipython/docker-notebook/tree/master/scipyserver
or https://github.com/jupyter/docker-demo-images::

  mkdir -p /var/projects/ipython
  docker run -d -p 8888:8888 -e "PASSWORD=MakeAPassword" -e "USE_HTTP=1" -v /var/projects/ipython/:/notebooks/ ipython/scipyserver


Current issue: neither connect to kernel. Investigate Beaker notebook.

Homework: https://ipython.org/ipython-doc/1/interactive/public_server.html

Future developments: spawn github-authenticated single-user servers on demand using

* https://github.com/jupyter/dockerspawner
* https://github.com/jupyter/oauthenticator
* https://github.com/jupyter/jupyterhub

Or use hosted service like https://www.dominodatalab.com/

Quantum GIS Server
==================
https://github.com/opengisch/docker-qgis-server-webclient
Run::
  mkdir /var/projects/qgis
  docker pull opengisch/qgis-server-webclient
  docker run -v /var/projects/qgis:/web -p 8100:80 -d -t opengisch/qgis-server-webclient

nginx conf::

  server {
    server_name qgis.DOMAIN.TLD;
    listen 80;
    location / {
      proxy_pass http://127.0.0.1:8100;
      proxy_redirect http://127.0.0.1:8100/ $scheme://$host/;
    }
  }

QGIS plugin idea: CKAN dataset shopping basket
==============================================
* CKAN API > filter by file type (WMS, WFS etc) > extract URLs and metadata > create .qgis project file
* Place .qgis project file into `/var/projects/qgis` which the qgis docker image mounts.
* Follow https://github.com/opengisch/docker-qgis-server-webclient.


Taverna
=======
Pull and run the `Taverna`_ Server `Docker image`_::

  docker pull taverna/taverna-server
  sudo docker run -p 8080:8080 -d taverna/taverna-server

.. _`Taverna`: http://www.taverna.org.uk/
.. _`Docker image`: https://hub.docker.com/r/taverna/taverna-server/


Map
===

This section will setup a local copy of NationalMap, which will come in handy for running behind firewalls, 
where the official NationalMap can't access datasets for preview.

1. Clone, install and run NationalMap as per `NationalMap docs`_::

  sudo apt-get install -y git-core gdal-bin
 
  curl -sL https://deb.nodesource.com/setup_0.12 | sudo bash -

  sudo apt-get install -y nodejs
  
  sudo npm install -g gulp
  
  /var/projects/$ clone https://github.com/NICTA/nationalmap.git
 
  /var/projects/nationalmap/$ sudo npm install
 
  /var/projects/nationalmap/$ gulp

2. Create a supervisord config ``/etc/supervisor/conf.d/nmap.conf``::
  
  [program:nmap]
  
  user=www-data
  
  stopasgroup=true
  
  autostart=true
  
  autorestart=true
  
  directory=/mnt/projects/nationalmap
  
  command=/usr/bin/npm start
  

3. Configure local data sources as required, e.g. `DPaW sources`_

4. Start NationalMap

  sudo supervisorctl stop nmap

  # modify datasources/xx.json

  /var/projects/nationalmap/$  gulp

  gulp && sudo supervisorctl start nmap

.. _`NationalMap docs`: https://github.com/NICTA/nationalmap/wiki/Deploying-a-copy-of-National-Map
.. _`DPaW sources`: https://github.com/datawagovau/nationalmap/tree/dpaw
