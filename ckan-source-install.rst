*******************
CKAN Source install
*******************

This chapter provides instructions to setup a CKAN instance from source inside an existing VM with special attention to managing storage needs.
An alternative installation is based on datacats / Docker. 
The datacats method is preferable over a source install for production use, while the source install allows more flexibility, e.g.
to use an existing database or SolR instance.

Generally, it follows the  CKAN source install at http://docs.ckan.org/en/latest/maintaining/installing/install-from-source.html and the Department of Parks and Wildlife's multi-tenant CKAN, see https://twitter.com/opendata/status/555760171017056256.

Install system dependencies
===========================

  pip install virtualenvwrapper
  apt-get install python-dev postgresql libpq-dev python-pip python-virtualenv git-core openjdk-7-jdk virtualenvwrapper \
  python-pastescript build-essential git redis-server postgresql-9.3-postgis-2.1 \
  libxml2-dev libxslt1-dev libgeos-c1 libjts-java apache2 libapache2-mod-wsgi libapache2-mod-rpaf unzip

Have virtualenvwrapper installed.

virtualenv
==========

As non-root user::

  mkproject ckan
  # with virtualenv "ckan" activated = workon ckan
  mkdir -p /var/projects/ckan/src
  cd /var/projects/ckan/src

Clone CKAN code from custom fork
================================

  add ssh public key to github profile
  git clone git@github.com:florianm/ckan.git
  git remote add upstream https://github.com/ckan/ckan.git
   
  git fetch upstream
  git merge upstream/master master -m 'merge upstream'
  git push

Database
========
Change data dir to external volume::

  service postgresql stop
  mkdir /var/www/pgdata
  mv /var/lib/postgresql/9.3/main/ /var/www/pgdata/
  ln -s /var/www/pgdata/main /var/lib/postgresql/9.3/main
  chown -R postgres:postgres /var/lib/postgresql/9.3/main
  service postgresql start

Setup database for CKAN::

  sudo -u postgres psql -l
  sudo -u postgres createuser -S -D -R -P ckan_default
  sudo -u postgres createuser -S -D -R -P -l datastore_default
  sudo -u postgres createdb -O ckan_default ckan_private -E utf-8
  sudo -u postgres createdb -O ckan_default ckan_public -E utf-8
  sudo -u postgres createdb -O ckan_default datastore_private -E utf-8
  sudo -u postgres psql -d ckan_private -f /usr/share/postgresql/9.3/contrib/postgis-2.1/postgis.sql
  sudo -u postgres psql -d ckan_private -f /usr/share/postgresql/9.3/contrib/postgis-2.1/spatial_ref_sys.sql
  sudo -u postgres psql -d ckan_private -c "ALTER TABLE spatial_ref_sys OWNER TO ckan_default;ALTER TABLE geometry_columns OWNER TO ckan_default;"

SolR
====

  cd /tmp
  sudo su
  wget http://archive.apache.org/dist/lucene/solr/4.10.2/solr-4.10.2.tgz
  tar -xvf solr-4.10.2.tgz
  cp -r solr-4.10.2/example /opt/solr
   
  cp /opt/solr/solr/collection1 /opt/solr/solr/ckan
  sed -i "s/collection1/ckan/g" /opt/solr/solr/ckan/core.properties
  ln -s /var/projects/ckan/src/ckan/ckan/config/solr/schema.xml /opt/solr/solr/ckan/conf/schema.xml

Download JTS-1.13, unpack and copy .jars to solr to enable spatial search::

  wget http://downloads.sourceforge.net/project/jts-topo-suite/jts/1.13/jts-1.13.zip
  unzip jts-1.13.zip
  cp *.jar /opt/solr/lib/
  cp *.jar /opt/solr/solr-webapp/webapp/WEB-INF/lib/

``/etc/supervisor/conf.d/solr.conf``::

  [program:solr]
  autostart=True
  autorestart=True
  directory=/opt/solr
  command=/usr/bin/java -Dsolr.solr.home=/opt/solr/solr -Djetty.logs=/opt/solr/logs -Djetty.home=/opt/solr -Djava.io.tmpdir=/tmp -jar /opt/solr/start.jar

Reload supervisorctl to start solr.

Test::

  curl 127.0.0.1:8983/solr/ckan/select/?fl=*,score&sort=score%20asc&q={!geofilt%20score=distance%20filter=true%20sfield=spatial_geom%20pt=42.56667,1.48333%20d=1}&fq=feature_code:PPL


Data store and datapusher
=========================
Follow the CKAN docs, but install datapusher into the project directory with::

  mkvirtualenv datapusher

This will create ``/var/projects/datapusher`` for files, and ``/var/venvs/datapusher`` for the virtualenv.


Spatial referencing
===================
Install ckanext-spatial, or use DPaW's special spatial widgets.
DPaW's widgets are live in action at their public data catalogue release candidate http://data-demo.dpaw.wa.gov.au/, 
and a video is available at https://vimeo.com/116324887.

Custom dataset schema
=====================
DPaW uses custom schemas, also live at their public data catalogue release candidate http://data-demo.dpaw.wa.gov.au/, but for simplicity's sake not enabled in the yes-we-ckan instance.


Multi-tenant install
====================
Follow the Department of Parks and Wildlife's guide  at https://twitter.com/opendata/status/555760171017056256.
Deployment diagram: http://data-demo.dpaw.wa.gov.au/dataset/da79b533-2813-42fb-a455-d9b32998abbf/resource/ae26b6f2-5ad5-46f1-b91d-3d40a19a77c4/download/CKANMultiSite.pdf

