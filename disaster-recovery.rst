*****************
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

Create a database snapshot from inside a datacats environment::

  (ckan)ubuntu@ip:/var/projects/ckan/SOURCE$ datacats paster db dump ../snapshot-DATE.sql
  
Notes:
* SOURCE is the datacats CKAN environment to be backed up
* DATE is the current datetime
* datacats paster db will run in the /ckan source directory, so the prefix ``../`` will create the file in the current directory
  

Backup & Restore
================

Users
-----
Ckanapi does not export users with password hashes. We'll have to migrate existin users manually.::

  sudo su
  sudo -u postgres pg_dump -a -O -t user -f user.sql ckan_private 
  
  # chown to datacats user and appropriate group
  sudo chown ubuntu:www-data user.sql
  
  # delete lines "default", "logged_in", "visitor" from user.sql

  # print passwords in datacats env:
  cat ~/.datacats/ENV_NAME/sites/primary/passwords.ini

  (datacats venv)me@machine:/path/to/ENV_NAME⟫ datacats shell
  
  #datacats shell
  (ckan)shell@ENV_NAME:~$ psql -h db -d ckan -U postgres -f user.sql
  # paste password for user "postgres"
  
  (ckan)shell@ENV_NAME:~$ exit


Groups, orgs, datasets
----------------------
Run after migrating users::

  ckanapi dump groups --all -p 3 -r http://source-ckan.org > groups.jsonl
  ckanapi dump organizations --all -p 3 -r http://source-ckan.org > organizations.jsonl
  ckanapi dump datasets --all -p 6 -r http://source-ckan.org > datasets.jsonl

  ckanapi load groups -p 3 -I groups.jsonl -r http://target-ckan.org -a API_KEY
  ckanapi load organizations -p 3 -I organizations.jsonl -r http://target-ckan.org -a API_KEY
  ckanapi load datasets -p 3 -I datasets.jsonl -r http://target-ckan.org -a API_KEY


Files
-----
Run any time::

  sudo rsync -Pavvr DATACATS_DATADIR/SOURCE/sites/primary/files/ DATACATS_DATADIR/TARGET/sites/primary/files/
  sudo chown -R ubuntu:www-data DATACATS_DATADIR/TARGET/sites/primary/files/

* The DATACATS_DATADIR/ defaults to ``~/.datacats``, but in our installation lives at ``/var/projects/datacats_data/``
* SOURCE and TARGET are two datacats CKAN environments
* ``datacats paster`` runs inside the datacats environment with prefix ``/project/`` for the current work dir
* http://target-ckan.org has an admin API_KEY



Archive
=======
Create a scheduled job to dump the db as well as the users, groups, orgs, datasets and 
rsync the dumps as well as the files (resource storage) to a secure place.


.. _`blog`: http://excess.org/article/2015/06/ckanapi-ckanext-scheming/