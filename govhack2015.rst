************
GovHack 2015
************
This project started out as one of several products entered by team *Bangers n' Mashup* for their project *yes we CKAN* at the GovHack 2015:

* The data portal demo `open-data.yes-we-ckan.org`_,
* a data science workbench (an R Studio Server) `rstudio.yes-we-ckan.org`_,
* an interactive application server (an R Shiny Server) `rshiny.yes-we-ckan.org`_, running
* an example app `timeseries explorer`_ (`ts code`_) visualizing open government data (one example dataset),
* and a whirlwind `video tour`_ of the whole stack.

.. _open-data.yes-we-ckan.org: http://open-data.yes-we-ckan.org/
.. _rstudio.yes-we-ckan.org: http://rstudio.yes-we-ckan.org/
.. _rshiny.yes-we-ckan.org: http://rshiny.yes-we-ckan.org/
.. _timeseries explorer: http://rshiny.yes-we-ckan.org/shiny-timeseries/
.. _ts code: https://github.com/florianm/shiny-timeseries
.. _video tour: https://vimeo.com/132628186

Disclaimer
==========
* The timeseries explorer was created before the GovHack and serves only as an illustration of an interactive, data-driven application to bridge the gap between raw data and meaningful visualisation.
* The datacats CKAN install was created after the GovHack, as a critical bug with a Docker version rendered datacats unusable during the contest, but was fixed one day afterwards.
* This documentation (submitted `version`_) was hosted on readthedocs.org after the contest and is under active development.
* The video was modified after the contest: the audio was shifted by ca. 200ms to counter the audio lag introduced by the hosting on vimeo.com, and a few words have been edited in the introductory slides. However, the general content and message have not been altered.

.. _submitted version: https://github.com/florianm/govhack2015/tree/7ff88829997058d01e4518592b208095acd88015/

Details
=======
The installation documented in `Alternatives` demonstrates the most low-level hands-on way of installing a CKAN data catalogue, an installation from source. 
There are several quicker, but less customisable ways:

* installation from package,
* installation from Docker image,
* using PaaS services like http://www.datacats.com/ or CKAN Galvanize http://datashades.com/ckan-galvanize/our-ckan-managed-platform/,
* contracting a third party to implement any of the above methods.

Our approach hopes to demonstrate our lessons learnt:

* It is possible to setup and host a working CKAN from scratch within a day.
* A source install, while not advisable for production use, allows to fix bugs and add new functionality, and contribute these improvements back to the CKAN community.
* We demonstrate how to scale the install to include and host other useful servers like R Studio Desktop and R Shiny Server.
* We include a scalable way to host a multi-tenant CKAN install following the WA Department of Parks and Wildlife's working `multi-tenant setup`_. The description of the multi-tenant install was created before the GovHack and is not to be considered for judging. However, we hope it provides value for real-world use past the GovHack event.

.. _multi-tenant setup: https://twitter.com/opendata/status/555760171017056256

Future directions
=================
This repository will be updated as work progresses on the CKAN installation. 
We plan to include installation examples using Datacats (plus Docker-based installs of R Studio Server and R Shiny Server) as well as Datashades' CKAN Galvanize.
