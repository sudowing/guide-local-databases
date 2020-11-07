

#  <a id="table-of-contents"></a>Table of Contents
* [PostGIS](#postgis)
    * [Create Docker Network & Run DB Container](#postgis-create_docker_network_run_db_container)
    * [Download Spatial Data (Shapefiles)](#postgis-download_spatial_data_shapefiles)
    * [Transform Spatial Data (Shapefiles) to SQL](#postgis-transform_spatial_data_shapefiles_to_sql)
        * [Install `shp2pgsql`](#postgis-transform_install_shp2pgsql)
        * [Use `shp2pgsql` to Produce SQL](#postgis-transform_use_shp2pgsql_to_produce_sql)
    * [Load the DB](#postgis-load_the_db)
    * [More on SRIDs](#postgis-more_on_sri_ds)
    * [Connect to DB Instance:](#postgis-connect_to_db_instance)
* [PostgreSQL (Dockerized DB Instance)](#postgresql_premade)
* [PostgreSQL (Simple ETL)](#postgresql)
    * [Download Sample Data](#postgresql-download_sample_data)
    * [Create Docker Network & Run DB Container](#postgresql-create_docker_network_run_db_container)
    * [Import Data via `psql` inside DB Container](#postgresql-import_data_via_psql_inside_db_container)
    * [Connect to DB Instance:](#postgresql-connect_to_db_instance)
* [MySQL](#mysql)
    * [Create Docker Network & Run DB Container](#mysql-create_docker_network_run_db_container)
    * [Load Sample Demo DB (`world-db`)](#mysql-load_sample_demo_db_world_db)
    * [Load Sample Demo DB (`sakila-db`)](#mysql-load_sample_demo_db_sakila_db)
    * [Connect to DB Instance:](#mysql-connect_to_db_instance)
* [SQLite](#sqlite)
    * [Download Sample DB (`chinook-db`)](#sqlite-download_sample_db_chinook_db)
    * [Connect to DB Instance:](#sqlite-connect_to_db_instance)

# <a id="postgis"></a> PostGIS

## <a id="postgis-create_docker_network_run_db_container"></a> Create Docker Network & Run DB Container
```sh
# create directories to hold persistantDB data
mkdir -p $(pwd)/volumes/postgis_db_data

# create directories to hold downloaded data
mkdir -p $(pwd)/volumes/shapefiles
mkdir -p $(pwd)/volumes/zip
mkdir -p $(pwd)/volumes/sql

# create docker network
docker network create mynetwork

# run mysql docker container
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

My system uses [DNF](https://en.wikipedia.org/wiki/DNF_(software)) as a package manager. yours may use yum, apt, brew or something else. If you need help, do a search with the string below (replacing the bracketed text with your OS version).

> Install shp2pgsql on `[insert OS version here]`

```sh
# install package containing `shp2pgsql` <- for .rpm-based distributions
sudo dnf install postgis-utils
```

### <a id="postgis-transform_use_shp2pgsql_to_produce_sql"></a> Use `shp2pgsql` to Produce SQL

Once you have `shp2pgsql` available locally -- you can use it to transform shapefiles (`.shp`) to SQL.

##### **NOTE:** The value passed with the `-s` flag is the [SRID](https://en.wikipedia.org/wiki/Spatial_reference_system). Always make an attempt to determine the `SRID` used in the source data, else the tool will fall back to a default that may not produce the projections you expect.

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
PGPASSWORD=secret

# execute SQL against a remote postgres instance
psql \
 -h localhost -d postgres\
 -U postgres \
 -f $(pwd)/volumes/sql/florida.gis_osm_places_free_1.sql

# keeping this here until I can confirm the multi line above works
# PGPASSWORD=secret psql \
#  -h localhost -d postgres\
#  -U postgres \
#  -f $(pwd)/volumes/sql/florida.gis_osm_places_free_1.sql
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

# <a id="postgresql"></a> PostgreSQL (Simple ETL)

The demo data used below was published by the team at [postgresqltutorial](https://www.postgresqltutorial.com/postgresql-sample-database/)

I've just added the various CLI steps needed to pull down this data and load it into a dockerized PostgreSQL instance.

## <a id="postgresql-download_sample_data"></a> Download Sample Data

The order of these steps differs from some of the other DB dialects because we want to mount the directory holding our SQL (`volumes/sql`) to the container running the DB. We are doing this so we can execute the import using the `psql` client already present in the container.

```sh
# create directory to hold persistant data & downloaded data
mkdir -p $(pwd)/volumes/psql_db_data
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

# run mysql docker container
docker run \
 -d \
 -v $(pwd)/volumes/db/psql_db_data:/var/lib/postgresql/data \
 -v $(pwd)/volumes/sql/dvdrental:/data \
 -e POSTGRES_PASSWORD=secret \
 -e POSTGRES_USER=dvdrental \
 --network mynetwork \
 --name dvd_rental_db \
 -p 5432:5432 postgres
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
**port**: 5432  
**database**: dvdrental  
**user**: dvdrental  
**password**: secret 

# <a id="mysql"></a> MySQL

## <a id="mysql-create_docker_network_run_db_container"></a> Create Docker Network & Run DB Container
```sh
# create directories to hold persistantDB data
mkdir -p $(pwd)/volumes/mysql_db_data

# create directories to hold downloaded data
mkdir -p $(pwd)/volumes/zip

# create docker network
docker network create mynetwork

# run mysql docker container
docker run -d \
 -v $(pwd)/volumes/mysql_db_data:/var/lib/mysql \
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
mkdir -p $(pwd)/volumes/sqlite_db_data

# create directories to hold downloaded data
mkdir -p $(pwd)/volumes/zip

# download demo db
curl https://raw.githubusercontent.com/lerocha/chinook-database/master/ChinookDatabase/DataSources/Chinook_Sqlite.sqlite \
 --output $(pwd)/volumes/sqlite_db_data/chinook.sqlite
```

## <a id="sqlite-connect_to_db_instance"></a> Connect to DB Instance:
>**db_location**: ./volumes/sqlite_db_data/chinook.sqlite
