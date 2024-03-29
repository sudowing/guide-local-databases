

#  <a id="table-of-contents"></a>Table of Contents
* [PostGIS](#postgis)
    * [Create Docker Network & Run DB Container](#postgis-create_docker_network_run_db_container)
    * [Download Spatial Data (Shapefiles)](#postgis-download_spatial_data_shapefiles)
    * [Transform Spatial Data (Shapefiles) to SQL](#postgis-transform_spatial_data_shapefiles_to_sql)
        * [Install `shp2pgsql`](#postgis-transform_install_shp2pgsql)
        * [Use `shp2pgsql` to Produce SQL](#postgis-transform_use_shp2pgsql_to_produce_sql)
    * [Load the DB](#postgis-load_the_db)
    * [More on SRIDs](#postgis-more_on_sri_ds)
    * [Connect to DB Instance](#postgis-connect_to_db_instance)
* [PostgreSQL (Dockerized DB Instance)](#postgresql_premade)
    * [Create Docker Network & Run DB Container](#postgresql_premade-create_docker_network_run_db_container)
    * [Connect to DB Instance](#postgresql_premade-connect_to_db_instance)
* [PostgreSQL (Simple ETL)](#postgresql)
    * [Download Sample Data](#postgresql-download_sample_data)
    * [Create Docker Network & Run DB Container](#postgresql-create_docker_network_run_db_container)
    * [Import Data via `psql` inside DB Container](#postgresql-import_data_via_psql_inside_db_container)
    * [Connect to DB Instance](#postgresql-connect_to_db_instance)
* [MySQL](#mysql)
    * [Create Docker Network & Run DB Container](#mysql-create_docker_network_run_db_container)
    * [Load Sample Demo DB (`world-db`)](#mysql-load_sample_demo_db_world_db)
    * [Load Sample Demo DB (`sakila-db`)](#mysql-load_sample_demo_db_sakila_db)
    * [Connect to DB Instance](#mysql-connect_to_db_instance)
* [SQLite](#sqlite)
    * [Download Sample DB (`chinook-db`)](#sqlite-download_sample_db_chinook_db)
    * [Connect to DB Instance](#sqlite-connect_to_db_instance)
* [SQL-Server (MSSQL)](#sql-server)
    * [Download Sample DB Backups](#mssql-download_sample_db_backups)
    * [Create Docker Volumes](#mssql-create_docker_volumes)
    * [Run SQL-Server in a Container](#mssql-run_sql_server_in_a_container)
    * [Connect to the DB](#mssql-connect_to_the_db)
    * [Restore Backups from DB client](#mssql-restore_backups_from_db_client)
* [Oracle](#oracle)
    * [Build Docker Image](#oracle-build_image-)
        * [Download Oracle Database Software](#oracle-build_image-download_oracle_software)
        * [Clone Oracle Image Repo](#oracle-build_image-clone_git_repo)
        * [Build Image via Script](#oracle-build_image-run_build_image_script)
    * [Run DB via Docker](#oracle-run_via_container)
    * [Load Sample Demo DB](#oracle-load_sample_schema)
        * [Download Oracle Client Dependencies](#oracle-load_sample_schema-download_dependencies)
        * [Download Sample Schemas](#oracle-load_sample_schema-download_schema)
        * [update ABSOLUTE PATH in scripts via perl](#oracle-load_sample_schema-update_references)
        * [mksample with passwords](#oracle-load_sample_schema-run_loader_script)

# <a id="postgis"></a> PostGIS

## <a id="postgis-create_docker_network_run_db_container"></a> Create Docker Network & Run DB Container
```sh
# create directories to hold persistantDB data
mkdir -p $(pwd)/volumes/db/postgis_db_data

# create directories to hold downloaded data
mkdir -p $(pwd)/volumes/shapefiles
mkdir -p $(pwd)/volumes/zip
mkdir -p $(pwd)/volumes/sql

# create docker network
docker network create mynetwork

# run postgres docker container (with postgis enabled)
docker run -d \
 -v $(pwd)/volumes/db/postgis_db_data:/var/lib/postgresql/data \
 -e POSTGRES_PASSWORD=secret \
 --network mynetwork \
 --name gis_db \
 -p 5432:5432 postgis/postgis
```
## <a id="postgis-download_spatial_data_shapefiles"></a> Download Spatial Data (Shapefiles)

The source for the GIS data to be loaded into PostGIS (for this guide at least) comes in the form of `shapefiles`.

You should have little trouble finding plenty of free shapefile data sets you can use as source data.

For this guide, you can pull down data for multiple states from this list of
[OpenStreetMap data for the U.S.](http://download.geofabrik.de/north-america/us.html). There are multiple formats available, but the process described below uses the compressed `.shp` files.

```sh
# download a target data set
curl http://download.geofabrik.de/north-america/us/florida-latest-free.shp.zip \
    --output $(pwd)/volumes/zip/florida-latest-free.shp.zip

# create a directory for the uncompressed loot
mkdir $(pwd)/volumes/shapefiles/florida

# uncompress downloaded files
unzip $(pwd)/volumes/zip/florida-latest-free.shp.zip \
    -d $(pwd)/volumes/shapefiles/florida

# remove compressed originals
rm $(pwd)/volumes/zip/florida-latest-free.shp.zip
```

## <a id="postgis-transform_spatial_data_shapefiles_to_sql"></a> Transform Spatial Data (Shapefiles) to SQL

Once you have the `.shp` files available locally, you use them to derive `.sql` files suitable for importing (via CLI) into the Dockerized PostGIS instance.

I believe there are multiple tools for completing this process -- but for this guide, we will use the [**shp2pgsql**](https://manpages.debian.org/stretch/postgis/shp2pgsql.1.en.html) dataloader. While this may eventually be put into a container, for now just install & run from your local machine.

### <a id="postgis-transform_install_shp2pgsql"></a> Install `shp2pgsql`

My system uses [DNF](https://en.wikipedia.org/wiki/DNF_(software)) as a package manager. Your system may use yum, apt, brew or something else entirely. If you need help, do a search with the string below (replacing the bracketed text with your OS version).

> Install shp2pgsql on `[insert OS version here]`

```sh
# install package containing `shp2pgsql` <- for .rpm-based distributions
sudo apt install postgis-utils

# https://computingforgeeks.com/how-to-install-postgis-on-ubuntu-debian/
```

### <a id="postgis-transform_use_shp2pgsql_to_produce_sql"></a> Use `shp2pgsql` to Produce SQL

Once you have `shp2pgsql` available locally -- you can use it to transform shapefiles (`.shp`) to SQL.

##### **NOTE 1:** I'm only processing a single shapefile (`gis_osm_places_free_1`), but you can use these steps to process all eighteen (18) present in the archive.

##### **NOTE 2:** The value passed with the `-s` flag is the [SRID](https://en.wikipedia.org/wiki/Spatial_reference_system). Always make an attempt to determine the `SRID` used in the source data, else the tool will fall back to a default that may not produce the projections you expect.

```sh
# transform shapefiles via shp2pgsql
shp2pgsql -s 4326 \
  $(pwd)/volumes/shapefiles/florida/gis_osm_places_free_1 > \
  $(pwd)/volumes/sql/florida.gis_osm_places_free_1.sql
```
##  <a id="postgis-load_the_db"></a> Load the DB

Once you've got `.sql` files, you can now execute them via CLI.

```sh
# set postgres password as ENV Variable
# and
# execute SQL against a remote postgres instance
PGPASSWORD=secret psql \
 -h localhost -d postgres\
 -U postgres \
 -f $(pwd)/volumes/sql/florida.gis_osm_places_free_1.sql
```
## <a id="postgis-more_on_sri_ds"></a> More on SRIDs

> **TL;DR:** You need to know the SRID to query correctly.

Many "Spatial Type" (`ST`) query functions that you will want to use to search through your new data will reference an [SRID](https://en.wikipedia.org/wiki/Spatial_reference_system) which is a numerical ID used to encode metadata about the geo data you loaded.

If you use the incorrect SRID in your queries, you'll get an error.

As such -- you can check the SRIDs for your imported records with this query and use that to ensure the SRID you're setting in the query terminal matches that data inside the db.

```sql
SELECT f_table_name, f_geometry_column, srid FROM geometry_columns;
```

## <a id="postgis-connect_to_db_instance"></a> Connect to DB Instance:
>**host**: localhost  
**port**: 5432  
**database**: postgres  
**user**: postgres  
**password**: secret  

# <a id="postgresql_premade"></a> PostgreSQL (Dockerized DB Instance)

##### [**Repo Link**](https://github.com/sudowing/cms-utilization-db)

A couple years ago, I published [a project that loads a PostgreSQL Instance](https://github.com/sudowing/cms-utilization-db). If you're just looking to get a sample PostgreSQL instance running, That's probably a decent project to use.

## <a id="postgresql_premade-create_docker_network_run_db_container"></a> Connect to DB Instance:

```sh
# create directories to hold persistantDB data
mkdir -p $(pwd)/volumes/db/postgres_cms_data

# run postgres docker container (with data pre-loaded)
docker run -d \
 -v $(pwd)/volumes/db/postgres_cms_data:/var/lib/postgresql/data \
 -e POSTGRES_DB=govdata \
 -e POSTGRES_USER=dbuser \
 -e PGPASSWORD=dbpassword \
 -e POSTGRES_PASSWORD=dbpassword \
 --network mynetwork \
 --name cms_db \
 -p 15433:5432 sudowing/cms-utilization-db
```

##### **NOTE 1:** Restoring this DB requires considerable disk. Ensure you've got 10G free.

##### **NOTE 2:** It will take several minutes (10+ mins) for the DB to be available after Docker container gets launched. There is data in the container and upon execution -- the startup calls an `init` process that starts the load. You can keep an eye on the Docker logs to get an idea where the process is.

```sh
docker logs cms_db
```

## <a id="postgresql_premade-connect_to_db_instance"></a> Connect to DB Instance:
>**host**: localhost  
**port**: 15433  
**database**: govdata  
**user**: dbuser  
**password**: dbpassword  

# <a id="postgresql"></a> PostgreSQL (Simple ETL)

The demo data used below was published by the team at [postgresqltutorial](https://www.postgresqltutorial.com/postgresql-sample-database/)

I've just added the various CLI steps needed to pull down this data and load it into a dockerized PostgreSQL instance.

## <a id="postgresql-download_sample_data"></a> Download Sample Data

The order of these steps differs from some of the other DB dialects because we want to mount the directory holding our SQL (`volumes/sql`) to the container running the DB. We are doing this so we can execute the import using the `psql` client already present in the container.

```sh
# create directory to hold persistant data & downloaded data
mkdir -p $(pwd)/volumes/db/psql_db_data
mkdir -p $(pwd)/volumes/zip
mkdir -p $(pwd)/volumes/sql

# download a target data set
curl https://sp.postgresqltutorial.com/wp-content/uploads/2019/05/dvdrental.zip \
    --output $(pwd)/volumes/zip/dvdrental.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/dvdrental.zip \
    -d $(pwd)/volumes/sql/dvdrental

# remove compressed originals
rm $(pwd)/volumes/zip/dvdrental.zip
```

## <a id="postgresql-create_docker_network_run_db_container"></a> Create Docker Network & Run DB Container

Here you will see we mount the directory holding our SQL (`volumes/sql`) to the container.

```sh
# create docker network
docker network create mynetwork

# run postgres docker container
docker run \
 -d \
 -v $(pwd)/volumes/db/psql_db_data:/var/lib/postgresql/data \
 -v $(pwd)/volumes/sql/dvdrental:/data \
 -e POSTGRES_PASSWORD=secret \
 -e POSTGRES_USER=dvdrental \
 --network mynetwork \
 --name dvd_rental_db \
 -p 15432:5432 postgres
```

## <a id="postgresql-import_data_via_psql_inside_db_container"></a> Import Data via `psql` inside DB Container

Run a `psql` process inside the container that references the mounted demo data.

```sh
# import demo data into db instance
docker exec \
    -i dvd_rental_db sh \
    -c 'exec psql -U dvdrental -d dvdrental -f /data/dvdrental.tar'
```

## <a id="postgresql-connect_to_db_instance"></a> Connect to DB Instance:
>**host**: localhost  
**port**: 15432  
**database**: dvdrental  
**user**: dvdrental  
**password**: secret 

# <a id="mysql"></a> MySQL

## <a id="mysql-create_docker_network_run_db_container"></a> Create Docker Network & Run DB Container
```sh
# create directories to hold persistantDB data
mkdir -p $(pwd)/volumes/db/mysql_db_data

# create directories to hold downloaded data
mkdir -p $(pwd)/volumes/zip

# create docker network
docker network create mynetwork

# run mysql docker container
docker run -d \
 -v $(pwd)/volumes/db/mysql_db_data:/var/lib/mysql \
 -e MYSQL_DATABASE=demo \
 -e MYSQL_USER=user \
 -e MYSQL_PASSWORD=password \
 -e MYSQL_ROOT_PASSWORD=secret \
 --network mynetwork \
 --name dev_mysql \
 -p 3306:3306 mysql:latest
```
## <a id="mysql-load_sample_demo_db_world_db"></a> Load Sample Demo DB (`world-db`)

```sh
# download demo db
curl https://downloads.mysql.com/docs/world_x-db.zip \
 --output $(pwd)/volumes/zip/world_x-db.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/world_x-db.zip \
    -d $(pwd)/volumes/zip/world_x-db

# remove compressed originals
rm $(pwd)/volumes/zip/world_x-db.zip

# import demo data into db instance
docker exec -i dev_mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < $(pwd)/volumes/zip/world_x-db/world_x-db/world_x.sql
```

## <a id="mysql-load_sample_demo_db_sakila_db"></a> Load Sample Demo DB (`sakila-db`)

```sh
# download demo db
curl https://downloads.mysql.com/docs/sakila-db.zip \
 --output $(pwd)/volumes/zip/sakila-db.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/sakila-db.zip \
    -d $(pwd)/volumes/zip/sakila-db

# remove compressed originals
rm $(pwd)/volumes/zip/sakila-db.zip

# import demo data into db instance (ddl)
docker exec -i dev_mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < $(pwd)/volumes/zip/sakila-db/sakila-db/sakila-schema.sql

# import demo data into db instance (data)
docker exec -i dev_mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < $(pwd)/volumes/zip/sakila-db/sakila-db/sakila-data.sql
```


## <a id="mysql-connect_to_db_instance"></a> Connect to DB Instance:
>**host**: localhost  
**port**: 3306  
**database**: demo  
**user**: user  
**password**: password  

# <a id="sqlite"></a> SQLite

## <a id="sqlite-download_sample_db_chinook_db"></a> Download Sample DB (`chinook-db`)

```sh
# create directories to hold persistantDB data
mkdir -p $(pwd)/volumes/db/sqlite_db_data

# create directories to hold downloaded data
mkdir -p $(pwd)/volumes/zip

# download demo db
curl https://raw.githubusercontent.com/lerocha/chinook-database/master/ChinookDatabase/DataSources/Chinook_Sqlite.sqlite \
 --output $(pwd)/volumes/db/sqlite_db_data/chinook.sqlite
```

## <a id="sqlite-connect_to_db_instance"></a> Connect to DB Instance:
>**db_location**: ./volumes/db/sqlite_db_data/chinook.sqlite

# <a id='sql-server'></a>SQL-Server (MSSQL)

## <a id='mssql-download_sample_db_backups'></a>Download Sample DB Backups

Microsoft published multiple sample databases for usage in training and development. You can download a few using the links below.
 - [adventureworks/AdventureWorksDW2019.bak](https://github.com/Microsoft/sql-server-samples/releases/download/adventureworks/AdventureWorksDW2019.bak)
 - [wide-world-importers-v1.0/WideWorldImportersDW-Full.bak](https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImportersDW-Full.bak)


```sh
# make the directory to hold the backups
mkdir -p ./volumes/mssql/backup

# download AdventureWorksDW2017
curl -L -o ./volumes/mssql/backup/AdventureWorksDW2017.bak https://github.com/Microsoft/sql-server-samples/releases/download/adventureworks/AdventureWorksDW2017.bak

# download WideWorldImportersDW
curl -L -o ./volumes/mssql/backup/WideWorldImportersDW-Full.bak 'https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImportersDW-Full.bak'
```

## <a id='mssql-create_docker_volumes'></a>Create Docker Volumes
```sh
docker volume create --driver local vlm_0001_mssql
docker volume create --driver local vlm_000_sqlserver
```

## <a id='mssql-run_sql_server_in_a_container'></a>Run SQL-Server in a Container
You will mount the two volumes created above for persisting DB data, and mount the directory holding the backups you downloaded to a location you can use to restore from the DB client.

```sh
docker run -d \
 -e "ACCEPT_EULA=Y" \
 -e "SA_PASSWORD=Alaska2017" \
 -p 21143:1433 \
 -v vlm_0001_mssql:/var/opt/mssql \
 -v vlm_000_sqlserver:/var/opt/sqlserver \
 -v $(pwd)/volumes/mssql/backup:/mssql_backups \
 --name mssql19 \
  mcr.microsoft.com/mssql/server:2019-latest
```

## <a id='mssql-connect_to_the_db'></a>Connect to the DB

```
host: localhost
user: sa
password: Alaska2017
```

## <a id='mssql-restore_backups_from_db_client'></a>Restore Backups from DB client
```sql

-- list all datafiles
RESTORE FILELISTONLY FROM DISK = '/mssql_backups/AdventureWorksDW2017.bak'

-- restore DB, explicitely moving datafiles
RESTORE DATABASE AdventureWorksDW2017 FROM DISK = '/mssql_backups/AdventureWorksDW2017.bak'
WITH
   MOVE 'AdventureWorksDW2017' to '/var/opt/mssql/data/AdventureWorksDW2017.mdf'
  ,MOVE 'AdventureWorksDW2017_log' to '/var/opt/mssql/data/AdventureWorksDW2017_log.mdf'

-- list all datafiles
RESTORE FILELISTONLY FROM DISK = '/mssql_backups/WideWorldImportersDW-Full.bak'

-- restore DB, explicitely moving datafiles
RESTORE DATABASE WideWorldImportersDW FROM DISK = '/mssql_backups/WideWorldImportersDW-Full.bak'
WITH
   MOVE 'WWI_Primary' to '/var/opt/mssql/data/WWI_Primary.mdf'
  ,MOVE 'WWI_UserData' to '/var/opt/mssql/data/WWI_UserData.mdf'
  ,MOVE 'WWI_Log' to '/var/opt/mssql/data/WWI_Log.mdf'
  ,MOVE 'WWIDW_InMemory_Data_1' to '/var/opt/mssql/data/WWIDW_InMemory_Data_1.mdf'
```

# <a id="#oracle"></a>Oracle

Running an Oracle DB locally via docker container is more complicated than the other DB dialects specifically because Oracle does not publish an image publically.

Instead, they provide guidance that can be used to build an image locally -- after the user downloads the software directly from their (login protected) software portal.

## <a id="#oracle-build_image"></a>Build Docker Image

In order to build an image you must (1) download software from their platform, (2) clone a git repo they have published, (3) move the downloaded artifact into a specific directory in the cloned repo and (4) run a script held in the repo.

Each step is detailed below.

### <a id="#oracle-build_image-download_oracle_software"></a>Download Oracle Database Software

Visit [Oracle Database Software Downloads](https://www.oracle.com/database/technologies/oracle-database-software-downloads.html) and download `19.3 - Enterprise Edition (also includes Standard Edition 2)` for
__Linux x86-64__.

> You must accept the Oracle License Agreement to download this software.

The downloaded filename should be:
`LINUX.X64_193000_db_home.zip`

### <a id="#oracle-build_image-clone_git_repo"></a>Clone Oracle Image Repo

```sh
git clone https://github.com/oracle/docker-images.git
```

### <a id="#oracle-build_image-run_build_image_script"></a>Build Image via Script

```sh
# cd to directory holding build script
cd docker-images/OracleDatabase/SingleInstance/dockerfiles

# move downloaded artifact to proper location
cp ~/Downloads/LINUX.X64_193000_db_home.zip 19.3.0/LINUX.X64_193000_db_home.zip

# build docker image (-t flag is the docker tag)
./buildContainerImage.sh \
    -v 19.3.0 \
    -t oracle/database:19.3.0-ee \
    -e

# wait...

# check to see new docker image
docker images | grep oracle/database
oracle/database                                 19.3.0-ee       8e1e74626c10   27 seconds ago   6.67GB
```

## <a id="#oracle-run_via_container"></a>Run DB via Docker

```sh
# create docker volume to persist db data
docker volume create oracle_db_data

# run newly created image mounting volume to persist data
docker run --rm -d \
    -p 1521:1521 \
    -p 5500:5500 \
    -e ORACLE_PDB=alpha \
    -e ORACLE_PWD=secret123 \
    -e ORACLE_EDITION=enterprise \
    -v oracle_db_data:/opt/oracle/oradata \
    --name oracle_db \
    oracle/database:19.3.0-ee

```

#### __NOTE__: There are several more `ENV VARs` that are supported beyond the minimal set implemented above. Details can be found [in the repo you cloned earlier](https://github.com/oracle/docker-images/tree/main/OracleDatabase/SingleInstance#running-oracle-database-enterprise-and-standard-edition-2-in-a-container).


## Connect to DB Instance:
>**host**: localhost  
**port**: 1521  
**database**: alpha  
**user**: system  
**password**: secret123  

## <a id="#oracle-load_sample_schema"></a>Load Sample Demo DB

Oracle provides several sample schemas for development/training purposes. The steps below walk users over how to load these schemas into an Oracle instance, but for more detail you can read [their documentation](https://docs.oracle.com/en/database/oracle/oracle-database/21/comsc/installing-sample-schemas.html#GUID-3820972A-08D7-4033-9524-1E36676594EE).


### <a id="#oracle-load_sample_schema-download_dependencies"></a>Download Oracle Client Dependencies

Oracle provides [many clients and tools](https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html), but we are going to only going to use the following for the rest of this guide:
- instantclient-basic
- instantclient-sqlplus
- instantclient-tools

```sh
# make directories to hold related files
mkdir -p $(pwd)/volumes/zip
mkdir -p $(pwd)/volumes/bin

# download basic instantclient-basic
curl https://download.oracle.com/otn_software/linux/instantclient/213000/instantclient-basic-linux.x64-21.3.0.0.0.zip \
    --output $(pwd)/volumes/zip/instantclient-basic-linux.x64-21.3.0.0.0.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/instantclient-basic-linux.x64-21.3.0.0.0.zip \
    -d $(pwd)/volumes/bin

# download instantclient-sqlplus
curl https://download.oracle.com/otn_software/linux/instantclient/213000/instantclient-sqlplus-linux.x64-21.3.0.0.0.zip \
    --output $(pwd)/volumes/zip/instantclient-sqlplus-linux.x64-21.3.0.0.0.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/instantclient-sqlplus-linux.x64-21.3.0.0.0.zip \
    -d $(pwd)/volumes/bin

# download instantclient-tools
curl https://download.oracle.com/otn_software/linux/instantclient/213000/instantclient-tools-linux.x64-21.3.0.0.0.zip \
    --output $(pwd)/volumes/zip/instantclient-tools-linux.x64-21.3.0.0.0.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/instantclient-tools-linux.x64-21.3.0.0.0.zip \
    -d $(pwd)/volumes/bin
```

### <a id="#oracle-load_sample_schema-download_schema"></a>Download Sample Schemas

```sh
# create directory to hold persistant data & downloaded data
mkdir -p $(pwd)/volumes/sql

# download a target data set
curl https://codeload.github.com/oracle/db-sample-schemas/zip/refs/tags/v19.2 \
    --output $(pwd)/volumes/zip/db-sample-schemas-19.2.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/db-sample-schemas-19.2.zip \
    -d $(pwd)/volumes/sql

# (optional) remove compressed originals
rm $(pwd)/volumes/zip/db-sample-schemas-19.2.zip
```

### <a id="#oracle-load_sample_schema-update_references"></a>update ABSOLUTE PATH in scripts via perl

```sh
# replace path in template sql
cd $(pwd)/volumes/sql/db-sample-schemas-19.2
perl -p -i.bak -e 's#__SUB__CWD__#'$(pwd)'#g' *.sql */*.sql */*.dat
```

### <a id="#oracle-load_sample_schema-run_loader_script"></a>mksample with passwords

```sh
# cd into dir holding instantclient bins
cd $(pwd)/volumes/bin/instantclient_21_3

# exec mksample script to load sample schemas
@/home/sudowing/Documents/repos/guide-local-databases/volumes/sql/db-sample-schemas-19.2/mksample secret123 secret123 secret123 secret123 secret123 secret123 secret123 secret123 EXAMPLE TEMP /tmp/oracle_demo_log localhost:1521/alpha
```

#### __NOTE 1__: Be sure to update the absolute path to `mksample` in the block above.
#### __NOTE 2__: `secret123` is the password that will be set for the various users that own schemas that get created. `EXAMPLE` + `TEMP` are default schemas set in the Oracle Instance. The last argument is the db connection string to your db.
