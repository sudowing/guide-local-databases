##### PostGIS Local Development Guide

The purpose of this repo is simply to document the steps needed to standup and load a local PostGIS instance.

Standing it up is simply running a docker container -- but loading it is a multi-step ETL process detailed below (download, convert, load)

### Run Local PostGIS Instance

PostGIS is an optional extention you can install in a Postgres DB. While you can take the minimal steps to enable it on an existing instance, running via a dedicated (and pre-configured) container is the easiest way to go.

```sh
docker network create mynetwork && \
docker run \
 -d \
 -v $(pwd)/volumes/db/db_data:/var/lib/postgresql/data \
 --network mynetwork \
 -e POSTGRES_PASSWORD=secret \
 --name gis_db \
 -p 5432:5432 postgis/postgis
```

### Download Shapefiles

The source for the GIS data to be loaded into PostGIS (for this guide at least) comes in the form of `shapefiles.

You should have little trouble finding plenty of free shapefile data sets you can use as source data.

For my local development, I pulled down data for multiple states from this list of
[OpenStreetMap data for the U.S.](http://download.geofabrik.de/north-america/us.html)

While I'm certain that the other format will work (else why would they exit right) -- the process described below is specific to compressed `.shp` files.

```sh
# download a target data set
curl http://download.geofabrik.de/north-america/us/florida-latest-free.shp.zip \
    --output $(pwd)/volumes/shapefiles/florida-latest-free.shp.zip

# create a directory for the uncompressed loot
mkdir $(pwd)/volumes/shapefiles/florida

# uncompress downloaded files
unzip $(pwd)/volumes/shapefiles/florida-latest-free.shp.zip \
    -d $(pwd)/volumes/shapefiles/florida

# remove compressed originals
rm $(pwd)/volumes/shapefiles/florida-latest-free.shp.zip
```

### Install `shp2pgsql`

The remaining heavy lift is in tranforming the `.shp` files into `.sql` files suitable for importing (via CLI) into the Dockerized PostGIS instance. For this guide, we will do this with the `shp2pgsql` dataloader.

While this may eventually be put into a container, with a `Dockerfile` in this repo, for now I'll just install & run from my local for time.

```sh
# install package containing `shp2pgsql`
sudo dnf install postgis-utils

# transform shapefiles via shp2pgsql magic
shp2pgsql -s 4326 \  #always set the SRID
  $(pwd)/volumes/shapefiles/florida/gis_osm_places_free_1 > \
  $(pwd)/volumes/sql/florida.gis_osm_places_free_1.sql
```

###  Import into DB

The `.sql` files produced by `shp2pgsql` are now ready to use. This is not different from running any other `.sql` file -- but since I always have to lookup the `psql` CLI syntax, I'll document below.

```sh
PGPASSWORD=secret psql \
 -h localhost -d postgres\
 -U postgres \
 -f $(pwd)/volumes/sql/florida.gis_osm_places_free_1.sql
```


### Note about SRIDs

Many "Spatial Type" (`ST`) query functions that you will want to use to search through your new data will reference an [SRID](https://en.wikipedia.org/wiki/Spatial_reference_system) which is a numerical ID used to encode metadata about the geo data you loaded.

If you use the incorrect SRID in your queries, you'll get an error.

As such -- you can check the SRIDs for your imported records with this query and use that to ensure the SRID you're setting in the query terminal matches that data inside the db.

```sql
SELECT f_table_name, f_geometry_column, srid FROM geometry_columns;
```


### Good Luck and Happy Mapping