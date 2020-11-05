

Databases
- PostgreSQL
- PostGIS
- MySQL
- Sqlite3










# MySQL


## create docker network & run DB
```

# create directory to hold persistant data & downloaded data
mkdir -p $(pwd)/volumes/mysql_db_data
mkdir -p $(pwd)/volumes/zip

# create docker network
docker network create mynetwork

# run mysql docker container
docker run \
 -d \
 -v $(pwd)/volumes/mysql_db_data:/var/lib/mysql \
 --network mynetwork \
 -e MYSQL_DATABASE=demo \
 -e MYSQL_USER=user \
 -e MYSQL_PASSWORD=password \
 -e MYSQL_ROOT_PASSWORD=secret \
 --name dev_mysql \
 -p 3306:3306 mysql:latest
```













## ooo

```

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






## ooo

```

# download demo db

curl https://downloads.mysql.com/docs/sakila-db.zip \
 --output $(pwd)/volumes/zip/sakila-db.zip

# uncompress downloaded files

unzip $(pwd)/volumes/zip/sakila-db.zip \
    -d $(pwd)/volumes/zip/sakila-db

# remove compressed originals

rm $(pwd)/volumes/zip/sakila-db.zip


# import demo data into db instance

docker exec -i dev_mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < $(pwd)/volumes/zip/sakila-db/sakila-db/sakila-schema.sql

docker exec -i dev_mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < $(pwd)/volumes/zip/sakila-db/sakila-db/sakila-data.sql


```



# SQLite


## llll

```

# create directory to hold persistant data & downloaded data
mkdir -p $(pwd)/volumes/sqlite_db_data
mkdir -p $(pwd)/volumes/zip
```


```

curl https://raw.githubusercontent.com/lerocha/chinook-database/master/ChinookDatabase/DataSources/Chinook_Sqlite.sqlite \
 --output $(pwd)/volumes/sqlite_db_data/chinook.sqlite



```







# PostgreSQL

```

# create directory to hold persistant data & downloaded data
mkdir -p $(pwd)/volumes/postgres_db_data
mkdir -p $(pwd)/volumes/zip
```




# PostGIS


```

# create directory to hold persistant data & downloaded data
mkdir -p $(pwd)/volumes/postgis_db_data
mkdir -p $(pwd)/volumes/shapefiles
mkdir -p $(pwd)/volumes/zip
mkdir -p $(pwd)/volumes/sql

```

## PostGIS Local Development Guide

The purpose of this repo is simply to document the steps needed to standup and load a local PostGIS instance.

Standing it up is simply running a docker container -- but loading it is a multi-step ETL process detailed below (download, convert, load)

## Run Local PostGIS Instance

PostGIS is an optional extention you can install in a Postgres DB. While you can take the minimal steps to enable it on an existing instance, running via a dedicated (and pre-configured) container is the easiest way to go.

```sh
docker network create mynetwork && \
docker run \
 -d \
 -v $(pwd)/volumes/db/postgis_db_data:/var/lib/postgresql/data \
 --network mynetwork \
 -e POSTGRES_PASSWORD=secret \
 --name gis_db \
 -p 5432:5432 postgis/postgis
```

## Download Shapefiles

The source for the GIS data to be loaded into PostGIS (for this guide at least) comes in the form of `shapefiles.

You should have little trouble finding plenty of free shapefile data sets you can use as source data.

For my local development, I pulled down data for multiple states from this list of
[OpenStreetMap data for the U.S.](http://download.geofabrik.de/north-america/us.html)

While I'm certain that the other format will work (else why would they exit right) -- the process described below is specific to compressed `.shp` files.

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

## Install `shp2pgsql`

The remaining heavy lift is in tranforming the `.shp` files into `.sql` files suitable for importing (via CLI) into the Dockerized PostGIS instance. For this guide, we will do this with the `shp2pgsql` dataloader.

While this may eventually be put into a container, with a `Dockerfile` in this repo, for now I'll just install & run from my local for time.

```sh
# install package containing `shp2pgsql`
sudo dnf install postgis-utils

# transform shapefiles via shp2pgsql magic
#always set the SRID
shp2pgsql -s 4326 \
  $(pwd)/volumes/shapefiles/florida/gis_osm_places_free_1 > \
  $(pwd)/volumes/sql/florida.gis_osm_places_free_1.sql
```


```
gis_osm_buildings_a_free_1
gis_osm_landuse_a_free_1
gis_osm_natural_a_free_1
gis_osm_natural_free_1
gis_osm_places_a_free_1
gis_osm_places_free_1
gis_osm_pofw_a_free_1
gis_osm_pofw_free_1
gis_osm_pois_a_free_1
gis_osm_pois_free_1
gis_osm_railways_free_1
gis_osm_roads_free_1
gis_osm_traffic_a_free_1
gis_osm_traffic_free_1
gis_osm_transport_a_free_1
gis_osm_transport_free_1
gis_osm_water_a_free_1
gis_osm_waterways_free_1
```

##  Import into DB

The `.sql` files produced by `shp2pgsql` are now ready to use. This is not different from running any other `.sql` file -- but since I always have to lookup the `psql` CLI syntax, I'll document below.

```sh
PGPASSWORD=secret psql \
 -h localhost -d postgres\
 -U postgres \
 -f $(pwd)/volumes/sql/florida.gis_osm_places_free_1.sql
```


## Note about SRIDs

Many "Spatial Type" (`ST`) query functions that you will want to use to search through your new data will reference an [SRID](https://en.wikipedia.org/wiki/Spatial_reference_system) which is a numerical ID used to encode metadata about the geo data you loaded.

If you use the incorrect SRID in your queries, you'll get an error.

As such -- you can check the SRIDs for your imported records with this query and use that to ensure the SRID you're setting in the query terminal matches that data inside the db.

```sql
SELECT f_table_name, f_geometry_column, srid FROM geometry_columns;
```


# Good Luck and Happy Mapping








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
