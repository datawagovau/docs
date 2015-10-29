**********************************
RStudio and RShiny package install
**********************************

This chapter describes the installation of R packages and servers to create a data science workbench.
Deployments using Docker, e.g. https://github.com/rocker-org/rocker, are easier to install,
however, using system users (e.g. AD users provided via winbind) for the RStudio user management requires a non-docker install.

Assuming the VM runs Ubuntu 14.04 "Trusty Tahr":

* add R repo to ``/etc/apt/sources.list``
* install system packagegs
* install R libraries
* download and install RStudio Server and RWhiny Server


  sudo su
  
  echo "deb http://cran.csiro.au/bin/linux/ubuntu trusty/" >> /etc/apt/sources.list
  
  apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E084DAB9
  apt-get update && apt-get -y upgrade
  apt-get -y dist-upgrade
  reboot
  
  apt-get -y install r-base r-base-dev gdebi-core texlive-full htop ncdu libcurl4-openssl-dev libxml2-dev
  
  sudo su - -c "R -e \"install.packages(c('shiny', 'shinyIncubator','markdown','whisker','Hmisc','dplyr','tidyr','lubridate','qcc','httr','RCurl','curl','devtools'), repos='http://cran.rstudio.com/');devtools::install_github("ropensci/ckanr")\""
  
  wget -O /tmp/rstudio.deb https://download2.rstudio.org/rstudio-server-0.99.467-amd64.deb
  gdebi -n /tmp/rstudio.deb
  wget -O /tmp/shiny.deb https://download3.rstudio.org/ubuntu-12.04/x86_64/shiny-server-1.4.0.721-amd64.deb
  gdebi -n /tmp/shiny.deb
  
  
  /home/ubuntu/shiny-serve$ git clone git@github.com:florianm/shiny-timeseries.git

Nginx
=====
Configure as in :doc:`workbench`.

Notes on Shiny's app dir:

* RStudio Server lets ubuntu access files only in own home directory ``/home/ubuntu``.
* RShiny Server can use any directory as app directory, default is ``/srv/shiny-server``.
* The external volume has enough space (100GB) and can be snapshotted (btrfs).
* The correct folders have already been created in the DevOps chapter.
* ``/srv/shiny-server`` is a symlink to ``/mnt/btrfsvol/shiny-server``.
* The user ubuntu's home directory contains another symlink to ``/srv/shiny-server``.