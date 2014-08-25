# doCOUNT DevOps
We use [ansible](https://github.com/ansible/ansible/) for infrastructure deployement.

## Installing ansible on OSX

1. Install libyaml

        $ brew install libyaml

2. Install python pacakages (need python > 2.6.1)

        $ sudo easy_install jinja2
        $ sudo easy_install PyYAML
        $ sudo easy_install paramiko

3. Install ansible (Not on homebrew as of time of writing)

        $ cd ~/github
        $ git clone git://github.com/ansible/ansible.git
        $ cd ./ansible
        $ source ./hacking/env-setup

4. Export hosts file

        $ export ANSIBLE_HOSTS=hosts

5. Test it's working
        $ ansible all -m ping -u butler

## Usage

Infrastructure is split up into playbooks which can be executed as follows:

        $ ansible-playbook playbooks/postgres.yml

To limit the execution of a playbook to specific hosts we can pass on an optional --limit paramater:

        $ ansible-playbook playbooks/postgres.yml --limit staging,demo


## Server Upgrade (DONE)
The sources for redis 2.6.1 and postrges 9.2 are packaged. However, the certificates still need to be added to apt.

- Postgres:

        $ wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -
- Redis:

        $ wget http://www.dotdeb.org/dotdeb.gpg
        $ cat dotdeb.gpg | sudo apt-key add -

## Database Upgrade (DONE)
The following steps are necessary to upgrade from 8.4 to 9.2:

        $ su - postgres
        $ pg_dumpall > dump.sql
        $ exit
        $ cp /var/lib/postgresql/dump.sql .
        $ aptitude purge postgresql-8.4
        $ apt-get -t squeeze-backports install libpq5 postgresql-common
        $ aptitude install postgresql-9.2
        $ su - postgres
        $ psql < ~/dump.sql

## Sidekiq hängt
The following steps will help to restart the sidekiq service

        $ stop bluepill
        $ kill the sidekiq service
        $ start bluepill
        $ ps -ef |grep sidekiq müsste jetzt 40 workers anzeigen wenn nicht lässt sich sidekiq auch wieder über capistrano neu starten
        $ cap myclimate sidekiq:restart


## Import DoCount Secret-Keys via Gpg
1. The first step for impor a encryptet database is to import the associated public and secret key. For importing the keys are the following keys neccessary

2. copy the public and secret key in your $HOME/.gpg folder

3. your .gpg folder should include now two files - info@docount.com.pub and secret_info@docount.key

4. first we import the public key via 
        $ gpg --import .gpg/info@docount.com.pub

5. in the next step import the private key with 

        $ gpg --allow-secret-key-import --import .gpg/secret_info@docount.key

6. your output for gpg --list-public-keys should look like

        $ pub   2048R/533AA2E2 2013-06-04
        $ uid                  doCOUNT <info@docount.com>
        $ sub   2048R/37B36963 2013-06-04

7. and your output for gpg --list-secret-keys should look like

        $ sec   2048R/533AA2E2 2013-06-04
        $ uid                  doCOUNT <info@docount.com>
        $ ssb   2048R/37B36963 2013-06-04

8. this allows us to decrypt a gpg encrypted file with

        $ gpg --output spms-myclimate.spms_production.2014.Tue.sql --decrypt spms-myclimate.spms_production.2014.Tue.sql.gpg


## Import actual database into development system
For this task it is neccessary that you have successfully imported the docount gpg keys

1. download the sql.gpg file from drive.google.com

2. diconnect all database connections

3. drop the database with in your rails project

        $ bundle exec rake db:drop:all

4. encrypt the file with 

        $ gpg --output  'name of the sql file'.sql --decrypt PATH_TO_YOUR.SQL.GPG

5. create the databases with 

        $ bundle exec rake db:create:all

6. as the sql file is a binary file we have to use a pg_restore

        $ pg_restore -c -d spms_myclimate -h localhost PATH_TO_YOUR_SQL_FILE

7. the output should include some messages about "cant find role spms" but this can we ignore generous

8. after the hopefully succesful import we have to check the search_path, for this we have to login to postgresql with
        $ psql -h localhost spms_myclimate

9. the search path 'show search_path;' should include myclimate, if not we have to set it with
        $SET SEARCH_PATH TO myclimate, public;

10. finally it is neccessary to update mapping of the tenant urls for development environments

11. login to rails console with:

        $ SCHEMA=myclimate bundle exec rails c

12. and update the urls with

        $ Tenant.all.each {|tenant| tenant.update_attribute("url", tenant.url.gsub(/\.[^\.]+$/, ".dev"))}


## COPY schema from one instance to other
This section describes how to copy data between different schemas

1. dump the schema as sql file

        $ pg_dump spms_production -h localhost -n 'SCHEMA_NAME_SOURCE' -U spms > SOURCE.sql

2. login to postgres

        $ \psql

3. delete the old target_schema if its exists with 

        $ DROP SCHEMA TARGET_SCHEMA_NAME CASCADE;

4. replace the ORIGINAL_SCHEMA_NAME with TARGET_SCHEMA_NAME$ in SOURCE.sql

5. import the updated SOURCE.sql into the new schema

        $ psql -h localhost spms_myclimate -f PATH/TO/SOURCE.sql

6. check and edit the search_paht if neccessary

        $ set search_path TO TARGET_SCHEMA_NAME, public


## POSSIBLE problems
Describe some problems a user of docount could face during his work

1. not enough database connections for opening a console

  $ if you get the message 'remaining connection slots are reserved for non-replication superuser connections (PG::Error)' we have to restore the connections

  $ Login as root user
