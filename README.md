

#  <a id="table-of-contents"></a>Table of Contents
* [postgis](#postgis)
    * [aaaa](#postgis-ssss)
    * [aaaa](#postgis-ssss)
    * [aaaa](#postgis-ssss)
    * [aaaa](#postgis-ssss)
    * [aaaa](#postgis-ssss)
* [postgres](#postgres)
    * [aaaa](#postgres-ssss)
    * [aaaa](#postgres-ssss)
    * [aaaa](#postgres-ssss)
    * [aaaa](#postgres-ssss)
    * [aaaa](#postgres-ssss)
* [mysql](#mysql)
    * [aaaa](#mysql-ssss)
    * [aaaa](#mysql-ssss)
    * [aaaa](#mysql-ssss)
    * [aaaa](#mysql-ssss)
    * [aaaa](#mysql-ssss)
* [sqlite](#sqlite)
    * [aaaa](#sqlite-ssss)
    * [aaaa](#sqlite-ssss)
    * [aaaa](#sqlite-ssss)
    * [aaaa](#sqlite-ssss)
    * [aaaa](#sqlite-ssss)






# PostGIS


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
## Download Spatial Data (Shapefiles)

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

## Transform Spatial Data (Shapefiles) to SQL

Once you have the `.shp` files available locally, you use them to derive `.sql` files suitable for importing (via CLI) into the Dockerized PostGIS instance.

I believe there are multiple tools for completing this process -- but for this guide, we will use the [**shp2pgsql**](https://manpages.debian.org/stretch/postgis/shp2pgsql.1.en.html) dataloader. While this may eventually be put into a container, for now just install & run from your local machine.

### Install `shp2pgsql`

My system uses [DNF](https://en.wikipedia.org/wiki/DNF_(software)) as a package manager. yours may use yum, apt, brew or something else. If you need help, do a search with the string below (replacing the bracketed text with your OS version).

> Install shp2pgsql on `[insert OS version here]`

```sh
# install package containing `shp2pgsql` <- for .rpm-based distributions
sudo dnf install postgis-utils
```

### Transform the Shapefiles to SQL

Once you have `shp2pgsql` available locally -- you can use it to transform shapefiles (`.shp`) to SQL.

##### **NOTE:** The value passed with the `-s` flag is the [SRID](https://en.wikipedia.org/wiki/Spatial_reference_system). Always make an attempt to determine the `SRID` used in the source data, else the tool will fall back to a default that may not produce the projections you expect.

```sh
# transform shapefiles via shp2pgsql
shp2pgsql -s 4326 \
  $(pwd)/volumes/shapefiles/florida/gis_osm_places_free_1 > \
  $(pwd)/volumes/sql/florida.gis_osm_places_free_1.sql
```


##  Load the DB

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
## More on SRIDs

> **TL;DR:** You need to know the SRID to query correctly.

Many "Spatial Type" (`ST`) query functions that you will want to use to search through your new data will reference an [SRID](https://en.wikipedia.org/wiki/Spatial_reference_system) which is a numerical ID used to encode metadata about the geo data you loaded.

If you use the incorrect SRID in your queries, you'll get an error.

As such -- you can check the SRIDs for your imported records with this query and use that to ensure the SRID you're setting in the query terminal matches that data inside the db.

```sql
SELECT f_table_name, f_geometry_column, srid FROM geometry_columns;
```

## You can now connect to the DB Instance:
>**host**: localhost  
**port**: 5432  
**database**: postgres  
**user**: postgres  
**password**: secret  

# MySQL


## create docker network & run DB
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


## Load Sample Demo DB (`world-db`)

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

## Load Sample Demo DB (`sakila-db`)

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


## You can now connect to the DB Instance:
>**host**: localhost  
**port**: 3306  
**database**: demo  
**user**: user  
**password**: password  

# SQLite


```sh
# create directories to hold persistantDB data
mkdir -p $(pwd)/volumes/sqlite_db_data

# create directories to hold downloaded data
mkdir -p $(pwd)/volumes/zip

# download demo db
curl https://raw.githubusercontent.com/lerocha/chinook-database/master/ChinookDatabase/DataSources/Chinook_Sqlite.sqlite \
 --output $(pwd)/volumes/sqlite_db_data/chinook.sqlite
```

## You can now connect to the DB Instance:
>**db_location**: ./volumes/sqlite_db_data/chinook.sqlite










# PostgreSQL


https://sp.postgresqltutorial.com/wp-content/uploads/2019/05/dvdrental.zip

```

# create directory to hold persistant data & downloaded data
mkdir -p $(pwd)/volumes/psql_db_data
mkdir -p $(pwd)/volumes/zip
mkdir -p $(pwd)/volumes/sql
```

## PostgreSQL Local Development Guide

The purpose of this repo is simply to document the steps needed to standup and load a local PostGIS instance.

Standing it up is simply running a docker container -- but loading it is a multi-step ETL process detailed below (download, convert, load)


## Download Sample DB

BLAH BLAH BLAH


```sh
# download a target data set

curl https://sp.postgresqltutorial.com/wp-content/uploads/2019/05/dvdrental.zip \
    --output $(pwd)/volumes/zip/dvdrental.zip


# uncompress downloaded files
unzip $(pwd)/volumes/zip/dvdrental.zip \
    -d $(pwd)/volumes/sql/dvdrental

# remove compressed originals
rm $(pwd)/volumes/zip/dvdrental.zip
```



## Run Local PostgreSQL Instance

BLAH BLAH BLAH

```sh
docker network create mynetwork && \
docker run \
 -d \
 -v $(pwd)/volumes/db/psql_db_data:/var/lib/postgresql/data \
 -v $(pwd)/volumes/sql/dvdrental:/data \
 --network mynetwork \
 -e POSTGRES_PASSWORD=secret \
 -e POSTGRES_USER=dvdrental \
 --name dvd_rental_db \
 -p 5432:5432 postgres
```


##  Import into DB via `pg_restore`

BLAH BLAH BLAH

```sh
                                   
# import demo data into db instance
docker exec \
    -i dvd_rental_db sh \
    -c 'exec psql -U dvdrental -d dvdrental -f /data/dvdrental.tar'

```



## PostGIS Local Development Guide

The purpose of this repo is simply to document the steps needed to standup and load a local PostGIS instance.

Standing it up is simply running a docker container -- but loading it is a multi-step ETL process detailed below (download, convert, load)