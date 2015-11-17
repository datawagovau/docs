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


Install extensions selectively
------------------------------

``datacats install`` installs all downloaded extensions in an environment directory.
To install an extension individually, run::

  datacats shell
  cd ckanext-EXTENSION
  python setup.py develop
  exit
  
This will install the extension into the datacats environment.
Running ``python setup.py develop`` outside the datacats shell will not install the extension into the datacats environment.
