**************
data.wa.gov.au
**************

.. image:: https://circleci.com/gh/florianm/govhack2015.svg?style=svg
  :target: https://circleci.com/gh/florianm/govhack2015
  :alt: 'Build status'
  
.. image:: https://badge.waffle.io/datawagovau/pandora.svg?label=ready&title=Ready 
  :target: https://waffle.io/datawagovau/pandora 
  :alt: 'Stories in Ready'

.. image:: https://readthedocs.org/projects/govhack2015/badge/?version=latest
  :target: http://govhack2015.readthedocs.org/en/latest/?badge=latest
  :alt: 'Documentation Status'

This repository contains instructions to setup and deploy a data portal based on CKAN, 
plus some infrastructure to automate the information pipelines along the pyramid of 
data - information - knowledge - wisdom.

.. image:: http://catalogue.alpha.data.wa.gov.au/dataset/1c91771c-8103-45db-9d49-47ea3615fffe/resource/d9ac2ab5-1092-41f4-8e1d-f07c8c149124/download/marine-science-data-flow---reproducible-reporting-using-ckan-o-sweave.jpeg
  :target: http://catalogue.alpha.data.wa.gov.au/dataset/data-wa-gov-au
  :alt: 'The Pyramid of Data - Information - Knowledge - Wisdom'


:doc:`deployment` documents the setup of a virtual machine, ready for the install of :doc:`ckan` and the :doc:`workbench`.
:doc:`govhack2015` contains the history of this project as a GovHack submission in July 2015.
The last two chapters illustrate an alternative source install for CKAN and the workbench packages. 
Installing from source is superseded by datacats and docker, but might be advatageous for systems with single-sign-on and shared authentication.


.. toctree::
  :maxdepth: 2

  deployment
  ckan
  workbench
  govhack2015
  ckan-source-install
  rstudio-package-install
  