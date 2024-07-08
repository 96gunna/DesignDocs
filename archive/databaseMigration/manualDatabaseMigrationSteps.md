# OED Manual Database Migration Steps

## Introduction

The following manual steps are meant to migrate OED's Postgres version across multiple major versions, specifically from 10.13 to 15.1. However, the steps should be generalizable to migrate across different versions or migrate different applications running Postgres in a Docker container. Before starting, make sure OED and its Docker containers are running.

## Changing the password encryption

Between Postgres versions 10.13 and 15.1, the default password encryption for Postgres users changed from md5 to scram-sha-256. As a result, the password encryption needs to be changed before migrating to 15.1. Note: It is possible to have a newer version of Postgres to still authenticate using md5, but this is not recommended as scram-sha-256 is a stronger hashing method which is more secure.

To change the encryption method from md5 to scram-sha-256, first go to the ``` postgresql.conf ``` file in the ``` postgres-data ``` folder. 

Then, search for the ``` password_encryption ``` parameter which should be set to md5 and change it to scram-sha-256. It should be around line 96. Be sure to uncomment the line by removing the hash (#) at the beginning of the line. 

Next, access the Postgres database as user oed. If in Visual Studio Code, this can be accomplished by opening the Docker extension, right clicking on ``` oed_database ```, selecting attach shell from the menu, and using the command ``` psql -U oed ``` in the terminal. Update the encryption method by running ``` SELECT pg_reload_conf(); ``` and check it has been updated with ``` SHOW password_encryption; ``` . 

In addition, the passwords of all users have to be set again using the new method. This can be done with the command ``` \password user_name ``` which will then prompt to enter the new password twice. At minimum, there should be two users, oed and postgres. Make sure to set the password to be the same as found in the ``` docker-compose.yml ``` file. The password for user postgres should match the ``` POSTGRES_PASSWORD ``` environment variable found around line 16 and the password for user oed should match the ``` OED_DB_PASSWORD ``` environment variable found around line 32.

## Creating a dump of the database

The next step is to create a dump of the database which will also act as a backup. In a terminal in the OED directory (not the database), run the command ``` docker compose exec database pg_dumpall --clean -U oed > dump_$(date +%Y-%m-%d"_"%H_%M_%S.sql) ``` . The variables will name the resulting dump file with the current date and time and the ``` --clean ``` file will include commands to drop the existing databases as OED will initialize the database with empty tables on a new installation. The ``` --clean ``` flag will also include commands to drop roles, however since OED will also automatically create roles on initialization, these commands are not necessary. Go into the dump file and comment out the lines under the drop roles heading where it drops roles oed and postgres, which should be around lines 24 and 25, and also the lines under the roles heading where it creates and alters roles oed and postgres, around lines 32-35. Now, the dump file is ready.

## Restarting OED with a new Postgres version

If using the same installation of OED, first stop OED from running. If in VSCode, close the remote connection. Then, delete the ``` oed_database ``` Docker container and delete the ``` postgres-data ``` folder from the OED directory. Next in the Dockerfile under ``` containers/database ```, change the ``` FROM postgres: ``` version from 10.13 to 15.1. Finally, reopen OED in container to bring it back up again.

Alternatively, if using a new installation of OED, simply just change the Postgres version in the Dockerfile.

After OED is running again, the Postgres version should now be 15.1, which can be checked by attaching a shell to the database container.

## Restoring the dump file

To put the contents of the dump into the new database, simply run the command ``` docker compose exec database psql -U postgres < dump_$(date +%Y-%m-%d"_"%H_%M_%S.sql) ``` with ``` dump_$(date +%Y-%m-%d"_"%H_%M_%S.sql) ``` being the file created from the ``` pgdump_all ``` from earlier. Note that the connection to the database this time is being made with user postgres instead of oed. This is necessary in order to drop the existing databases created when OED initialized the new database. After the command finishes running, the migration is complete.
