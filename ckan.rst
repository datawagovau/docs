****
CKAN
****

This chapter documents a CKAN install using `Datacats`_. Features:

* Datacats will be installed from source inside a virtualenv.
* The virtualenv will live in ``/var/venvs/ckan``.
* The datacats installation and environments will live in ``/var/projects/ckan``.
* The datacats data dir ``~/.datacats`` is symlinked to ``/mnt/btrfsvol/datacats_data``.
* The directory ``/var/lib/docker`` contains all docker images.
* The directories ``/var/venvs``, ``/var/projects`` and ``/var/lib/docker`` are symlinked to the external 100 GB volume ``/mnt/btrfsvol/``.
* Nginx will be configured to reverse-proxy custom subdomain to servers running on local ports.
* The domain hosting redirects requests to the custom subdomains to the VM's static IP.

A note on conflicting pip and requests packages: 
If pip gets ``ImportError: cannot import name IncompleteRead``, run ``sudo easy_install requests==2.2.1``.
To avoid this bug, we'll install datacats (and every other python-based project) into its own virtualenv,
where they can have their preferred requests version, and the system can have its own, pip-compatible version (e.g. requests==2.2.1).


.. _`Datacats`: http://www.datacats.com/

Directories and symlinks
========================
With virtualennvwrapper installed and sourced from ``~/.bashrc``, create virtualenv and project directories for datacats::
  
  mkproject ckan

With our custom settings, this will create ``/var/projects/ckan`` as project directory, and ``/var/venvs/ckan`` for the virtualenv.
It will also enable the virtualenv. Deactivate and reactivate to use the virtualenv's binaries rather than the system-wide ones.
Create a symlink to ``~/.datacats`` *before* any datacats environment is created. Otherwise, ``~/.datacats`` will contain files owned by 
other users (root, postgres) and will have to be moved by the root user and chowned to the current user, while all datacats environments are stopped.


Datacats install
================
With the datacats virtualenv activated, clone the datacats repo and pull the Docker images::

  workon ckan
  (ckan)ubuntu@ip-172-31-14-85:/var/projects/ckan$ 
  
  git clone https://github.com/datacats/datacats.git
  cd datacats
  python setup.py install
  datacats pull -a


Datacats environments
=====================
Create an environment as per datacats docs::

  (ckan)ubuntu@ip:/var/projects/ckan$ 
  datacats create --ckan latest --site-url http://catalogue.alpha.data.wa.gov.au datawagovau 5000

This will create ``/var/projects/ckan/datawagovau``,
install ckan and run the server on the given port (here: 5000).

Reverse proxy the datacats environment
======================================
If the environment runs on e.g. port 5000, add this section to ``/etc/nginx/sites-enabled/base.conf``
to host the environment on a subdomain::

  proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=cache:30m max_size=250m;
  proxy_temp_path /tmp/nginx_proxy 1 2;
  
  server {
      server_name catalogue.alpha.data.wa.gov.au;
      listen 80;
      client_max_body_size 2G;
      location / {
          proxy_pass http://127.0.0.1:5000;
          proxy_set_header X-Forwarded-For $remote_addr;
          proxy_set_header Host $host;
          proxy_cache cache;
          proxy_cache_bypass $cookie_auth_tkt;
          proxy_no_cache $cookie_auth_tkt;
          proxy_cache_valid 30m;
          proxy_cache_key $host$scheme$proxy_host$request_uri;
      }
  }


Test and apply with ``sudo nginx configtest`` and ``sudo service nginx reload``.
This will create a working CKAN without any further extensions. To enable the extensions, follow the next chapter.

Extensions
==========
The following list of extensions displays their installation status on our example `CKAN`_.

.. _CKAN: http://catalogue.alpha.data.wa.gov.au/

The installation process is:

* installed: extension repo is downloaded and installed into the datacats environment
* active: extension is enabled in CKAN config
* working: extension actually works

Between the last two steps lies a varying amount of configuration to the environment, including but not limited to:

* database additions, 
* running of servers (celery task queue, redis message queue, pycsw server etc.),
* addition of config files (pycsw, harvester), 
* writing to weird and wonderful locations outside the installation directory (flickrapi being the worst offender).

All these additions have to be applied within the contraints of datacats' docker-based deployment approach.

===========================  =============================================  ====================
Extension                    Functionality                                  Status
===========================  =============================================  ====================
`ckanext-dcat`_              Metadata export as RDF                         working           
`ckanext-pages`_             Static pages                                   working           
`ckanext-spatial`_           Georeferencing (DPaW widget), spatial search   fork working  
`ckanext-scheming`_          Custom metadata schema                         fork working  
`ckanext-pdfview`_           PDF resource preview                           working           
`ckanext-geoview`_           Spatial resource preview                       working
`ckanext-cesiumpreview`_     NationalMap preview                            working
`ckanext-harvest`_           Metadata harvesting                            in dev, currently scripted 
`pycsw`_                     CSW endpoint for CKAN                          working
`ckan-galleries`_            Image hosting on CKAN                          some issues       
`ckanext-doi`_               DOI minting                                    in dev
`ckanext-archiver`_          Resource file archiving                        working
`ckanext-qa`_                QA checks (e.g. has DOI)                       working
`ckanext-hierarchy`_         Hierarchical organisations                     working           
`WA data licenses`_          WA data licensing                              pending license list  
`ckanext-geopusher`_         SHP and KML to GeoJSON converter               working
`ckanext-featuredviews`_     Showcase resource views                        works in layout 1
`ckanext-showcase`_          Replace featured items                         working
`ckanext-disqus`_            User comments                                  working
`ckanext-datawagovautheme`_  Data.wa.gov.au theme                           working           
`ckanapi`_                   Python client for CKAN API                     working           
`ckanR`_                     R client for CKAN API                          working           
===========================  =============================================  ====================

.. _ckanext-dcat: https://github.com/ckan/ckanext-dcat
.. _ckanext-pages: https://github.com/datawagovau/ckanext-pages
.. _ckanext-spatial: https://github.com/ckan/ckanext-spatial
.. _ckanext-scheming: https://github.com/open-data/ckanext-scheming
.. _ckanext-pdfview: https://github.com/ckan/ckanext-pdfview
.. _ckanext-geoview: https://github.com/ckan/ckanext-geoview
.. _ckanext-harvest: https://github.com/ckan/ckanext-harvest
.. _pycsw: https://github.com/geopython/pycsw
.. _ckan-galleries: https://github.com/DataShades/ckan-galleries
.. _ckanext-doi: https://github.com/NaturalHistoryMuseum/ckanext-doi
.. _ckanext-archiver: https://github.com/ckan/ckanext-archiver
.. _ckanext-qa: https://github.com/ckan/ckanext-qa
.. _ckanext-hierarchy: https://github.com/datagovuk/ckanext-hierarchy
.. _`WA data licenses`: http://ands.org.au/publishing/licensing.html
.. _ckanapi: https://github.com/ckan/ckanapi
.. _ckanR: http://extensions.ckan.org/extension/ckanr/
.. _ckanext-geopusher: https://github.com/datacats/ckanext-geopusher
.. _ckanext-featuredviews: https://github.com/datacats/ckanext-featuredviews
.. _ckanext-datawagovautheme: https://github.com/datawagovau/ckanext-datawagovautheme
.. _`ckanext-cesiumpreview`: https://github.com/datagovau/ckanext-cesiumpreview
.. _`ckanext-disqus`: https://github.com/ckan/ckanext-disqus
.. _`ckanext-showcase`: https://github.com/ckan/ckanext-showcase

Note: Unless specified otherwise, all code examples are executed as 
non-root user "ubuntu" (who must be in the docker group) in the CKAN environment's directory, e.g.::

  workon ckan
  (ckan)ubuntu@ip:/var/projects/ckan/
  # cd into datacats environment "test"
  cd test/
  (ckan)ubuntu@ip:/var/projects/ckan/test$

Download extensions
===================
Run::

  git config --global push.default matching 
  
  datacats install
  
  # ckanext-spatial custom fork
  git clone git@github.com:datawagovau/ckanext-spatial.git
  cd ckanext-spatial
  git remote add upstream https://github.com/ckan/ckanext-spatial.git
  git fetch upstream
  git merge upstream/master master -m 'merge upstream'
  git push
  cd ..
  
  # ckanext-scheming custom fork
  git clone git@github.com:florianm/ckanext-scheming.git
  cd ckanext-scheming
  git remote add upstream https://github.com/open-data/ckanext-scheming.git
  git fetch upstream
  git merge upstream/master master -m 'merge upstream'
  git push
  cd ..
  
  #git clone https://github.com/ckan/ckanext-pages.git
  git clone https://github.com/datawagovau/ckanext-pages.git
  # git clone https://github.com/ckan/ckanext-harvest.git
  git clone git@github.com:datawagovau/ckanext-harvest.git
  git clone https://github.com/ckan/ckanext-dcat.git
  git clone https://github.com/geopython/pycsw.git
  git clone https://github.com/ckan/ckanext-geoview.git
  git clone https://github.com/datagovau/ckanext-cesiumpreview.git
  git clone https://github.com/ckan/ckanext-pdfview.git
  git clone https://github.com/ckan/ckanext-archiver.git
  git clone https://github.com/ckan/ckanext-qa.git
  git clone https://github.com/datagovuk/ckanext-hierarchy.git
  git clone https://github.com/NaturalHistoryMuseum/ckanext-doi.git
  #git clone https://github.com/DataShades/ckan-galleries.git
  git clone https://github.com/ckan/ckanapi.git
  git clone https://github.com/datacats/ckanext-geopusher.git
  git clone https://github.com/datacats/ckanext-featuredviews.git
  git clone https://github.com/datawagovau/ckanext-datawagovautheme.git
  git clone https://github.com/ckan/ckanext-disqus.git
  git clone https://github.com/ckan/ckanext-showcase.git
  
  # pycsw dependencies
  sudo apt-get install -y python-dev libxml2-dev libxslt-dev libgeos-dev

Manage dependency conflicts
===========================
Before running through this section, note that dependency conflicts are caused by
multiple independently developed code bases of ckan and its plugins.
Each code base pins third party library versions known to work at the time of release.
Naturally, the most established extensions, e.g. spatial and harvesting, have the
oldest dependencies, while brand new extensions, e.g. agls, require much newer
libraries.

Note: currently, the setup works without this section.

Review possible collisions at http://rshiny.yes-we-ckan.org/ckan-pip-collisions/.
Note, the following example lists dependencies current as of October 2015 and will outdate quickly.
We recommend to research your own version conflicts and use this example as a how-to guide,
but with your own dependencies.
In our example the following packages have differing, hard-coded requirements::

  grep -rn --include="*requirements*" 'requests' .
  grep -rn --include="*requirements*" 'six' .
  grep -rn --include="*requirements*" 'lxml' .
  grep -rn --include="*requirements*" 'python-dateutil' .
  grep -rn --include="*requirements*" 'SQLAlchemy' .

We'll need to update all colliding requirement versions to one that works across all extensions.
In our case, a simple bump to the highest mentioned version will work, such as with the perfectly backwards compatible ``requests`` library.
In other cases, breaking changes between different dependency versions could require an upgrade to an actual extension.

Batch-modify version numbers as shown here work on our listed extensions at the time of writing.
Modify to your actual needs. Warning - a mistake in this step could corrupt your installed code (including CKAN source),
requiring to ``git checkout`` incorrectly modified files in each repo.::

  grep -rl --include="*requirements*" 'requests' . | xargs sed -i 's/^.*requests.*$/requests==2.7.0/g'
  grep -rl --include="*requirements*" 'six' . | xargs sed -i 's/^.*six^.*/six==1.9.0/g'
  grep -rl --include="*requirements*" 'lxml' . | xargs sed -i 's/^.*lxml^.*/lxml==3.4.4/g'
  grep -rl --include="*requirements*" 'python-dateutil' . | xargs sed -i 's/^.*python-dateutil^.*/python-dateutil==2.4.2/g'
  grep -rl --include="*requirements*" 'SQLAlchemy' . | xargs sed -i 's/^.*SQLAlchemy.*$/SQLAlchemy==0.9.6/g'
  
  # review version numbers
  grep -rn --include="*requirements*" 'requests' .
  grep -rn --include="*requirements*" 'six' .
  grep -rn --include="*requirements*" 'lxml' .
  grep -rn --include="*requirements*" 'python-dateutil' .
  
  # any other requirements conflicts?
  cat `find . -name '*requirements*'` | sort | uniq
  
  
To fix issues with any dependency versions::
  
  datacats shell
  pip freeze | grep lchemy
  pip install SQLAlchemy==0.9.6
  exit

E.g., this is necessary when receiving this error on datacats reload::

  File "/usr/lib/ckan/local/lib/python2.7/site-packages/geoalchemy2/comparator.py", line 52, in <module>
  class BaseComparator(UserDefinedType.Comparator):
  AttributeError: type object 'UserDefinedType' has no attribute 'Comparator'
  Starting subprocess with file monitor

  
Install extensions
==================
To install all extensions and their dependencies in the site's environment, run::

  datacats install


Modify datacats containers
==========================
Some extensions require modifications to the database, or additional servers, such as a message queue (redis) or a task runner (celery).
Following `ckanext-spatial docs`_ and `ckanext-harvest docs`_ with datacats' `paster`_ command::

  # (re)install postgis, add redis
  datacats tweak --install-postgis
  datacats tweak --add-redis
  # datacats tweak --add-pycsw # soon
  datacats reload
  # pulls redis image
  
  # initdb for spatial
  cd ckanext-spatial
  datacats paster spatial initdb
  cd ..
  
  # initdb for harvester, plus two celery containers, see also below
  cd ckanext-harvest
  datacats paster harvester initdb
  datacats paster -d harvester gather_consumer
  datacats paster -d harvester fetch_consumer
  cd ..
  

.. _`ckanext-spatial docs`: http://docs.ckan.org/projects/ckanext-spatial/en/latest/install.html#configuration
.. _`ckanext-harvest docs`: https://github.com/ckan/ckanext-harvest/blob/master/README.rst
.. _`paster`: http://docs.datacats.com/commands.html#paster

Note: ``git init`` the theme extension (ckanext-SITEtheme) to preserve significant customisations.

Enable extensions
=================
General procedure:

* Edit config `vim development.ini`, replace the Plugins section with settings below
* Apply changes with `datacats reload`. That should be it!

``development.ini``::

  ## Authorization Settings
  ckan.auth.anon_create_dataset = false
  ckan.auth.create_unowned_dataset = false
  ckan.auth.create_dataset_if_not_in_organization = false
  ckan.auth.user_create_groups = false
  ckan.auth.user_create_organizations = false
  ckan.auth.user_delete_groups = false
  ckan.auth.user_delete_organizations = false
  ckan.auth.create_user_via_api = true
  ckan.auth.create_user_via_web = true
  ckan.auth.roles_that_cascade_to_sub_groups = admin editor member

  ckan.cors.origin_allow_all = true
  
  ## Plugins Settings
  base = cesium_viewer resource_proxy datastore datapusher datawagovau_theme stats archiver qa pages featuredviews showcase disqus
  sch = scheming_datasets
  rcl = recline_grid_view recline_graph_view recline_map_view
  prv = text_view image_view recline_view pdf_view webpage_view
  geo = geo_view geojson_view
  spt = spatial_metadata spatial_query geopusher
  hie = hierarchy_display hierarchy_form
  dcat = dcat dcat_rdf_harvester dcat_json_harvester dcat_json_interface
  hrv = harvest ckan_harvester csw_harvester
  ckan.plugins = %(base)s %(sch)s %(rcl)s %(prv)s %(dcat)s %(geo)s %(spt)s %(hrv)s %(hie)s
  
  ckanext.geoview.ol_viewer.formats = wms wfs gml kml arcgis_rest gft
  ckan.views.default_views = cesium_view %(prv)s geojson_view
  
  ckan.max_resource_size = 1000000
  ckan.max_image_size = 200000
  ckan.resource_proxy.max_file_size = 31457280
  
  # ckanext-scheming
  scheming.dataset_schemas = ckanext.datawagovautheme:datawagovau_dataset.json
  #scheming.organization_schemas = ckanext.datawagovautheme:datawagovau_organization.json

  # ckanext-harvest
  ckan.harvest.mq.type = redis
  ckan.harvest.mq.hostname = redis
  ckanext.spatial.harvest.continue_on_validation_errors= True
  
  # ckanext-pages
  ckanext.pages.organization = True
  ckanext.pages.group = True
  # disable to make space for static pages:
  ckanext.pages.about_menu = True
  ckanext.pages.group_menu = True
  ckanext.pages.organization_menu = True
  
  # ckanext-disqus
  # add Engage to site > add a subaccount to your disqus account for this CKAN
  # choose name = disqus.name
  # settings > advanced >
  # add %(site_url)s to trusted domains, e.g. catalogue.beta.data.wag.gov.au
  disqus.name = datawagovau-ckan
  
  ckan.datapusher.formats = csv xls xlsx tsv application/csv application/vnd.ms-excel application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
  
  ckan.activity_streams_enabled = true
  ckan.activity_list_limit = 31
  
  # Front-end settings
  ...
  
  ## Internationalisation Settings
  ckan.locale_default = en_AU
  ckan.locale_order = en_AU ...
  
  # google recaptcha: your secret credentials


PyCSW
=====
While our contribution is in development, we'll manually build and run a dockerised pycsw
using our `datacats fork`_::

  cd /var/projects/ckan/datacats/docker/pycsw/
  docker build -t datacats/pycsw .
  docker run -d -p 9000:8000 -it datacats/pycsw python /var/www/pycsw/csw.wsgi

.. _datacats fork: https://github.com/datawagovau/datacats/tree/pycsw-docker

This will build a pycsw server image with harvesting enabled (transactions) for non-local IPs 
and run a pycsw server on localhost:9000. 
See also nginx settings in :doc:`deployment` to expose the csw server publicly.


******************
Disaster Recovery
*****************

Examples
========

The following examples are taken from Ian Ward's talk at the CKANConf Ottawa 2015 as published on his `blog`_.
Exporting data (backup), restoring to same (disaster recovery) or remote (migration) instance::

  ckanapi dump groups | ssh otherbox ckanapi load groups -p 3
  ckanapi dump organizations | ssh otherbox ckanapi load organizations -p 3
  ckanapi dump datasets | ssh otherbox ckanapi load datasets -p 3
  
Or pulling data from remote into local CKAN::

  ckanapi dump datasets -r http://sourceckan | ckanapi load datasets -p 3
  
Track metadata using git::

  ckanapi dump datasets > datasets.jsonl
  git diff datasets.jsonl --stat
  datasets.jsonl | 52 ++++++++++++++++++++++++++++++++++++----------------
  1 file changed, 36 insertions(+), 16 deletions(-)
  
Summaries::

  head -5 datasets.jsonl | jq .title
  jq 'select(.organization.name!="nrcan-rncan")' -c datasets.jsonl | wc -l
  

Distributed loading::

  split -n l/3 datasets.jsonl part
  ckanapi load datasets -r http://web1 -a ... < partaa &
  ckanapi load datasets -r http://web2 -a ... < partab &
  ckanapi load datasets -r http://web3 -a ... < partac &
  

Backup
======
Create a database snapshot from inside a datacats environment::

  (ckan)ubuntu@ip:/var/projects/ckan/SOURCE$ datacats paster db dump ../snapshot-DATE.sql
  
Notes:
* SOURCE is the datacats CKAN environment to be backed up
* DATE is the current datetime
* datacats paster db will run in the /ckan source directory, so the prefix ``../`` will create the file in the current directory

Restore
=======
Copy the database snapshot into the new environment, clean the target db, and load the snapshot::
  
  rsync -Pavvr DATACATS_DATADIR/SOURCE/sites/primary/files/ DATACATS_DATADIR/TARGET/sites/primary/files/
  rsync -Pavvr /var/projects/ckan/SOURCE/snapshot-DATE.sql /var/projects/ckan/TARGET/
  (ckan)ubuntu@ip:/var/projects/ckan/TARGET$ datacats paster db clean
  (ckan)ubuntu@ip:/var/projects/ckan/TARGET$ datacats paster db load /project/snapshot-DATE.sql
  
Notes:
* The DATACATS_DATADIR/ defaults to ``~/.datacats``, but in our installation lives at ``/var/projects/datacats_data/``
* SOURCE and TARGET are two datacats CKAN environments
* ``datacats paster`` runs inside the datacats environment with prefix ``/project/`` for the current work dir

Archive
=======
Create a scheduled job to dump the db and rsync the db snapshot as well as the files to a secure storage.


.. _`blog`: http://excess.org/article/2015/06/ckanapi-ckanext-scheming/

**********
Operations
**********

The following sections illustrate operational use of extensions after successful installation.


Quality Assurance (ckanext-qa)
==============================
Create a supervisor script::

  # run the celery daemon (stops with each datacats reload)
  datacats paster -d celeryd
  
  # run the qa update
  cd ckanext-qa
  datacats paster qa update
  cd ..

Geo-referencing a dataset (ckanext-spatial)
===========================================
Watch `this video`_ to get started.

.. _this video: https://vimeo.com/116324887

Adding a static page (ckanext-pages)
====================================
Click on the pages icon, add content, save.


CKAN to CKAN Harvesting (ckanext-harvest)
=========================================
If there's no custom ckanext-scheming metadata schema enabled, plain CKAN harvesting will work.

Create a harvest source at ``/harvest`` with 

* url ``http://SOURCE.yes-we-ckan.org``,
* source type ``CKAN``,
* update frequency ``always``,
* configuration::

  {"api_version": 1,
  "default_extras":{"url":"{harvest_source_url}/dataset/{dataset_id}"},
  "override_extras": "false",
  "user":"local-sysadmin-username",
  "read_only": true,
  "remote_groups": "create",
  "remote_orgs": "create",
  "clean_tags":"true",
  "override_extras": true}

* save.

On the shell inside the ckanext-harvest extension's folder::

  ckanext-harvest$ datacats paster -d harvester gather_consumer
  ckanext-harvest$ datacats paster -d harvester fetch_consumer
  ckanext-harvest$ datacats paster harvester job-all
  ckanext-harvest$ datacats paster harvester run

This will run the two gather/fetch-consumer celery containers in the background (-d flag),
reharvest the source (job-all does the same as clicking "reharvest" on every data source),
and run the harvest job.

Note: The two daemonised consumer containers will have to be restarted manually after each ``datacats reload``.

Known problem: The source and target CKAN instances must have the exact same schema,
either the default, or the exact same custom schema. Otherwise, the "extra" fields 
will collide with ckanext-scheming's "extra" fields, with an error such as:

  nature-reserves-proposals-beeliar-wetlands-park-sw-corridor-wl19850899

  Invalid package with GUID nature-reserves-proposals-beeliar-wetlands-park-sw-corridor-wl19850899: 
  {'extras': [{'key': ['There is a schema field with the same name']}, {'key': ['There is a schema field with the same name']}]}

Notably, a custom extra field "spatial", or a scheming field "spatial" will also not work.

Harvesting between catalogues with custom CKAN dataset schemas
==============================================================
Note: This section describes (only partly implemented) functionality under development.

Custom metadata schemas via ckanext-scheming can be harvested using our fork of ckanext-harvest.

Create a harvesting job, e.g. on catalogue.alpha.data.wa.gov.au:

* Url http://landgate.alpha.data.wa.gov.au/
* Source type: CKAN (we overwrote the ckanharvester)
* Update frequency: manual
* Configuration: (for ckanext-datawagovautheme:datawagovau_dataset.json)::

  {
   "api_version": 1,
   "user":"florianm",
   "remote_groups": "create",
   "remote_orgs": "create",
   "clean_tags":false,
   "force_all":true,
   "disable_extras":true,
   "default_fields":{
    "data_portal":"{harvest_source_url}",
    "landing_page":"{harvest_source_url}/dataset/{dataset_id}"
   },
    "field_mapping":{
    "doi": "doi",
    "citation": "citation",
    "published_on": "published_on",
    "last_updated_on": "last_updated_on",
    "update_frequency": "update_frequency",
    "data_temporal_extent_begin": "data_temporal_extent_begin",
    "data_temporal_extent_end": "data_temporal_extent_end",
    "spatial": "spatial"
   }
  }

* Organization: landgate (exists on catalogue.alpha)

Then run inside the ckanext-harvest extension folder::

  (ckan)ubuntu@ip:/var/projects/ckan/datawagovau/ckanext-harvest$
  datacats paster -d harvester gather_consumer
  datacats paster -d harvester fetch_consumer
  datacats paster harvester job-all
  datacats paster harvester run

* Every ``datacats reload`` requires a restart of the gather and fetch consumer.
* ``job-all`` is the shortcut for clicking "Reharvest" on all harvest sources.
* ``run`` starts all pending jobs, and refreshes the status of running jobs.


Harvesting WMS
==============
We will harvest metadata from a WMS GetCapabilities statement into a local pycsw server,
then harvest that pycsw server into CKAN using the CSW harvester.

Create a file `test-wms.xml` for the official pycsw example WMS::

  <?xml version="1.0" encoding="UTF-8"?>
  <Harvest 
    xmlns="http://www.opengis.net/cat/csw/2.0.2" 
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xsi:schemaLocation="http://www.opengis.net/cat/csw/2.0.2 http://schemas.opengis.net/csw/2.0.2/CSW-publication.xsd" 
    service="CSW" version="2.0.2">
    <Source>http://webservices.nationalatlas.gov/wms/1million</Source>
    <ResourceType>http://www.opengis.net/wms</ResourceType>
    <ResourceFormat>application/xml</ResourceFormat>
  </Harvest>


Landgate's SLIP Classic WMS, e.g. `slip_classic.xml`::

  <?xml version="1.0" encoding="UTF-8"?>
  <Harvest xmlns="http://www.opengis.net/cat/csw/2.0.2" 
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xsi:schemaLocation="http://www.opengis.net/cat/csw/2.0.2 http://schemas.opengis.net/csw/2.0.2/CSW-publication.xsd" 
    service="CSW" version="2.0.2">
    <Source>http://srss-dev.landgate.wa.gov.au/slip/wmspublic.php</Source>
    <ResourceType>http://www.opengis.net/wms</ResourceType>
    <ResourceFormat>application/xml</ResourceFormat>
  </Harvest>

In the pycsw folder, run::

  (datacats)ubuntu@ip:/var/projects/datacats/public/pycsw$ 
  python bin/pycsw-admin.py -c post_xml -u http://localhost:9000/pycsw/csw.py -x test-wms.xml
  python bin/pycsw-admin.py -c post_xml -u http://localhost:9000/pycsw/csw.py -x slip_classic.xml

to receive::

  Initializing static context
  Executing HTTP POST request test-wms.xml on server http://localhost:8086/pycsw/csw.py
  <?xml version="1.0" encoding="UTF-8" standalone="no"?>
  <!-- pycsw 2.0-dev -->
  <csw:HarvestResponse xmlns:csw30="http://www.opengis.net/cat/csw/3.0" xmlns:fes20="http://www.opengis.net/fes/2.0" xmlns:ows11="http://www.opengis.net/ows/1.1" xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:inspire_common="http://inspire.ec.europa.eu/schemas/common/1.0" xmlns:atom="http://www.w3.org/2005/Atom" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:dct="http://purl.org/dc/terms/" xmlns:ows20="http://www.opengis.net/ows/2.0" xmlns:ows="http://www.opengis.net/ows" xmlns:apiso="http://www.opengis.net/cat/csw/apiso/1.0" xmlns:gml="http://www.opengis.net/gml" xmlns:dif="http://gcmd.gsfc.nasa.gov/Aboutus/xml/dif/" xmlns:xlink="http://www.w3.org/1999/xlink" xmlns:gco="http://www.isotc211.org/2005/gco" xmlns:gmd="http://www.isotc211.org/2005/gmd" xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#" xmlns:srv="http://www.isotc211.org/2005/srv" xmlns:ogc="http://www.opengis.net/ogc" xmlns:fgdc="http://www.opengis.net/cat/csw/csdgm" xmlns:inspire_ds="http://inspire.ec.europa.eu/schemas/inspire_ds/1.0" xmlns:csw="http://www.opengis.net/cat/csw/2.0.2" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:os="http://a9.com/-/spec/opensearch/1.1/" xmlns:soapenv="http://www.w3.org/2003/05/soap-envelope" xmlns:sitemap="http://www.sitemaps.org/schemas/sitemap/0.9" xsi:schemaLocation="http://www.opengis.net/cat/csw/2.0.2 http://schemas.opengis.net/csw/2.0.2/CSW-publication.xsd"><csw:TransactionResponse version="2.0.2"><csw:TransactionSummary><csw:totalInserted>21</csw:totalInserted><csw:totalUpdated>0</csw:totalUpdated><csw:totalDeleted>0</csw:totalDeleted></csw:TransactionSummary><csw:InsertResult><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8</dc:identifier><dc:title>1 Million Scale WMS Layers from the National Atlas of the United States</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-ports1m</dc:identifier><dc:title>1 Million Scale - Ports</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-national1m</dc:identifier><dc:title>1 Million Scale - National Boundary</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-elevation</dc:identifier><dc:title>1 Million Scale - Elevation 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-impervious</dc:identifier><dc:title>1 Million Scale - Impervious Surface 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-coast1m</dc:identifier><dc:title>1 Million Scale - Coastlines</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-cdl</dc:identifier><dc:title>1 Million Scale - 113th Congressional Districts</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-states1m</dc:identifier><dc:title>1 Million Scale - States</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-naturalearth</dc:identifier><dc:title>1 Million Scale - Natural Earth Shaded Relief 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-elsli0100g</dc:identifier><dc:title>1 Million Scale - Color-Sliced Elevation 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-landcov100m</dc:identifier><dc:title>1 Million Scale - Land Cover 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-cdp</dc:identifier><dc:title>1 Million Scale - 113th Congressional Districts by Party</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-amtrak1m</dc:identifier><dc:title>1 Million Scale - Railroad and Bus Passenger Stations</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-airports1m</dc:identifier><dc:title>1 Million Scale - Airports</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-svsri0100g</dc:identifier><dc:title>1 Million Scale - Satellite View with Shaded Relief 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-one_million</dc:identifier><dc:title>1 Million Scale WMS Layers from the National Atlas of the United States</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-satvi0100g</dc:identifier><dc:title>1 Million Scale - Satellite View 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-srcoi0100g</dc:identifier><dc:title>1 Million Scale - Color Shaded Relief 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-srgri0100g</dc:identifier><dc:title>1 Million Scale - Gray Shaded Relief 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-treecanopy</dc:identifier><dc:title>1 Million Scale - Tree Canopy 100 Meter Resolution</dc:title></csw:BriefRecord><csw:BriefRecord><dc:identifier>urn:uuid:711ab330-4b63-4d24-90cc-185b6a701cd8-landwatermask</dc:identifier><dc:title>1 Million Scale - Land/Water Mask 100 Meter Resolution</dc:title></csw:BriefRecord></csw:InsertResult></csw:TransactionResponse></csw:HarvestResponse>
  Done


In CKAN, create a harvest source (e.g. through the GUI at ``/harvest``) 
following the `harvest docs`_:

* url: http://pycsw.beta.data.wa.gov.au
* Source type: CSW Server
* Update frequency: Daily (creates a new harvest job at given frequency, which has to be run by `datacats paster harvester run`)
* Configuration::

  {"default_tags":["wms_harvested"],
  "clean_tags":true,
  "default_groups":["test"],
  "remote_groups":"create",
  "override_extras": false,
  "read_only":true,
  "force_all":true}
  
.. _harvest docs: https://github.com/ckan/ckanext-harvest

Notes on configuration:

* ``clean_tags`` makes tags url-safe
* ``override_extras`` does not influence whether remote extras will be created locally (which will fail if ckanext-scheming is installed)
* ``force_all`` will update even unchanged datasets

Custom WMS harvesting
=====================
The following example is a real-life use case of harvesting GeoServer 1.8 WMS/WFS endpoints
into our customised dataset schema ``datawagovau_dataset.json``.

Challenges:

* Custom WMS with authentication for publicly available layers (you right that read)
* We harvest a proxy which provides our (secret) credentials to access the public WMS/WFS layers
* Custom conventions for extracting groups and organizations from WMS and from assumptions
* Custom methods to extract dataset title, dataset ID and publication date from WMS layer name

This harvester contains too many unique assumptions and requires context, and the WMS endpoint will be superseded soon.
Therefore, it is implemented as iPython Notebook, with the functions and operations cleanly separated.
This brings as benefits:

* faster turn-around (Shift-Enter) than customising extension and going through redis queues
* can still be refactored into an extension when appropriate
* self-documenting
* separates confidential credentials into separate config file
* shapshots online at http://catalogue.alpha.data.wa.gov.au/dataset/slip-harvesting
* follows harvester logic of building dicts from harvested source, and uploading dicts to CKAN

The following sections contain work in progress on not yet working extensions.

Mint DOIs (ckanext-doi)
=======================

* Fork ckanext-doi
* Register with ANDS to mint DOI at http://ands.org.au/services/cmd-registration.html
* Customise ckanext-doi to support both (existing) Datacite and (add) ANDS minting
* CKAN config parameters

Media gallery (ckan-galleries)
==============================
Add ``dfmp`` to plugins. Results:

* dfmp theme overrides datacats theme (undesirable, submitted as ckan-galleries #5 at https://github.com/DataShades/ckan-galleries/issues/5)
* one of the layout options crashes ckan
* ``import``ing a flickr pool does not seem to have any effect (needs investigation)
* twitter account is hard-coded and blocked due to over-use

Will use as soon as bugs are fixed upstream.


Harvesting WMS notes
====================

* VicRoads AGO http://vicroadsopendata.vicroadsmaps.opendata.arcgis.com/data.json
* Dcat example https://raw.githubusercontent.com/ckan/ckanext-dcat/master/examples/dataset.json
* Meteoswiss harvester example https://github.com/ogdch/ckanext-meteoswiss

Setup ckanext-spatial spatial harvesters::

  cd ckanext-spatial
  datacats paster ckan-pycsw setup -p ../pycsw.cfg
  cd ..
  
  cd pycsw
  python csw.wsgi &
  cd ..
  
  
  http://127.0.0.1:PORT/?service=CSW&version=2.0.2&request=GetCapabilities


[pycsw docs](http://geopython.github.io/pycsw-workshop/docs/intro/intro-exercises.html#metadata-harvesting)

Author: Keith Moss, Landgate WA::

  python pycsw/bin/pycsw-admin.py -c post_xml -u http://localhost:8000/pycsw/csw.py -x pycsw/post.xml
  
  # Woo! PyCSW harvested the example service!
  
  # Got this error from lxml:
  # http://docs.ckan.org/projects/ckanext-spatial/en/latest/install.html#when-running-the-spatial-harvesters
  
  
  # So, continue_on_validation_errors wasn't being picked up from production.ini
  
  vim ckanext-spatial/ckanext/spatial/harvesters/base.py
  # Just commented out the block that aborts
  
  
  # Harvesting SLIP Classic
  nano pycsw/post-slip-classic.xml
  
  python pycsw/bin/pycsw-admin.py -c post_xml -u http://localhost:8000/pycsw/csw.py -x pycsw/post-slip-classic.xml
  
  curl -x https://USER:PASS@www2.landgate.wa.gov.au/ows/wmspublic -X POST -d post-slip-classic.xml http://localhost:8000/pycsw/csw.py
  
  vim pycsw/post-firewatch.xml
  python pycsw/bin/pycsw-admin.py -c post_xml -u http://localhost:8000/pycsw/csw.py -x pycsw/post-firewatch.xml
  
  # List records
  # http://127.0.0.1:PORT/?request=GetRecords&service=CSW&version=2.0.2&resultType=results&outputSchema=http://www.isotc211.org/2005/gmd&typeNames=csw:Record&elementSetName=summary
  
  # Get a record
  # http://127.0.0.1:PORT/?outputFormat=application%2Fxml&service=CSW&outputSchema=http%3A%2F%2Fwww.isotc211.org%2F2005%2Fgmd&request=GetRecordById&version=2.0.2&elementsetname=full&id=urn:uuid:634b1df3-cc33-4f9b-89b1-0f04ed41a208-treecanopy
  
  # Dumb reverse proxy to Classic to work around the "pycsw doesn't do secured services" thing
  # http://gis.stackexchange.com/questions/103191/getting-error-after-trying-to-harvest-from-geoserver-into-pycsw
  python pycsw/bin/pycsw-admin.py -c post_xml -u http://localhost:PORT/pycsw/csw.py -x pycsw/post-slip-classic-rp.xml

CSW ISO19139 XML validation issues (pyCSW?) https://github.com/ngds/ckanext-ngds/issues/442
CSIRO on CKAN harvesting https://www.seegrid.csiro.au/wiki/Infosrvices/CKANHarvestingGuide

data.wa.gov.au theming extension
================================
This section documents the additions we made to the theming extension beyond the obvious CSS overrides.
If you have installed the extension, the following steps are already done.

First, we copied a few templates we needed to override::

  mkdir -p ckanext-datawagovautheme/ckanext/datawagovautheme/templates/package/
  mkdir -p ckanext-publictheme/ckanext/datawagovautheme/templates/organization/
  cp ckan/ckan/templates/organization/read_base.html ckanext-datawagovautheme/ckanext/datawagovautheme/templates/organization/
  cp ckan/ckan/templates/package/search.html ckanext-datawagovautheme/ckanext/datawagovautheme/templates/package/
  cp ckan/ckan/templates/package/read_base.html ckanext-datawagovautheme/ckanext/datawagovautheme/templates/package/

Fix CKAN organisation activity stream bug (now fixed upstream):

* ``vim cckanext-datawagovautheme/ckanext/datawagovautheme/templates/organization/read_base.html``
* Add "offset=0" to  as per  ckan/ckan#2466 https://github.com/ckan/ckan/issues/2466::

  {{ h.build_nav_icon('organization_activity', _('Activity Stream'), id=c.group_dict.name, offset=0) }}


Add spatial search (ckanext-spatial):

* Enable the  spatial search widget http://docs.ckan.org/projects/ckanext-spatial/en/latest/spatial-search.html#spatial-search-widget:
* ``vim ckanext-datawagovautheme/ckanext/datawagovautheme/templates/package/search.html``

Add to block ``secondary_content``::

  {% snippet "spatial/snippets/spatial_query.html", default_extent="[[-40,110], [0, 130]]" %}


CSS fixes should be obsolete once our spatial widget (with fixed libraries and CSS) gets merged.
To render the dataset search icon visible, add to the CKAN CSS::

  .leaflet-draw-draw-rectangle, .leaflet-draw-draw-polygon {height:22px; width:22px;}


Add spatial preview (ckanext-spatial):

* If *not* using the custom dataset schema (which we are in this example, 
enable the `dataset extent map`_:
* ``vim ckanext-datawagovautheme/ckanext/datawagovautheme/templates/package/read_base.html``
* Add to block ``secondary_content``::

  {% set dataset_extent = h.get_pkg_dict_extra(c.pkg_dict, 'spatial', '') %}
  {% if dataset_extent %}
    {% snippet "spatial/snippets/dataset_map_sidebar.html", extent=dataset_extent %}
  {% endif %}

.. _dataset extent map: http://docs.ckan.org/projects/ckanext-spatial/en/latest/spatial-search.html#dataset-extent-map

Contribute to Datacats
=======================
This section is not part of the setup workflow and can be skipped.

Update a local docker image
---------------------------
To update a datacats image locally, as reported at https://github.com/datacats/datacats/issues/210::

  $ docker run -datacats/web apt-get install -y whatever
  $ docker ps -lq
  6f51fba7febb
  $ docker commit 6f51fba7febb datacats/web

This image will be used until you do a datacats pull the next time. 
You can do run/commit as many times as you'd like, but if you're making a lot of changes you'll want to change the actual Dockerfiles in the source and rebuild them::

  cd datacats/docker
  docker build -t datacats/web . 

Report a bug
------------
Submit working additions as a new datacats issue at https://github.com/datacats/datacats/issues/new.