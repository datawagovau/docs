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
  
  #git clone https://github.com/datawagovau/ckanext-datawagovautheme.git
  git clone git@github.com:datawagovau/ckanext-datawagovautheme.git
  
  #git clone https://github.com/ckan/ckanext-pages.git
  git clone https://github.com/datawagovau/ckanext-pages.git
  
  # git clone https://github.com/ckan/ckanext-harvest.git
  git clone git@github.com:datawagovau/ckanext-harvest.git
  
  git clone https://github.com/ckan/ckanext-archiver.git
  git clone https://github.com/datagovau/ckanext-cesiumpreview.git
  git clone https://github.com/ckan/ckanext-dcat.git
  git clone https://github.com/ckan/ckanext-disqus.git
  git clone https://github.com/NaturalHistoryMuseum/ckanext-doi.git
  git clone https://github.com/datacats/ckanext-featuredviews.git
  #git clone https://github.com/DataShades/ckan-galleries.git
  git clone https://github.com/ckan/ckanext-geoview.git
  git clone https://github.com/datacats/ckanext-geopusher.git
  git clone https://github.com/datagovuk/ckanext-hierarchy.git
  git clone https://github.com/ckan/ckanext-pdfview.git
  git clone https://github.com/ckan/ckanext-qa.git
  git clone https://github.com/ckan/ckanext-showcase.git
  
  git clone https://github.com/ckan/ckanapi.git
  git clone https://github.com/geopython/pycsw.git

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
