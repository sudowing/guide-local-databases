# postgis-guide


# run local DB
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

# download shapefiles

http://download.geofabrik.de/north-america/us.html

```sh
curl http://download.geofabrik.de/north-america/us/florida-latest-free.shp.zip \
    --output $(pwd)/volumes/shapefiles/florida-latest-free.shp.zip
```

```sh
mkdir $(pwd)/volumes/shapefiles/florida
unzip $(pwd)/volumes/shapefiles/florida-latest-free.shp.zip \
    -d $(pwd)/volumes/shapefiles/florida
rm $(pwd)/volumes/shapefiles/florida-latest-free.shp.zip
```

## install shp2pgsql (eventually in docker container)
```sh
sudo dnf install postgis-utils

```

```sh
sudo dnf install postgis-utils
```

## convert shapefiles to .sql for import into postgis enabled db
```sh
shp2pgsql $(pwd)/volumes/shapefiles/florida/gis_osm_buildings_a_free_1 > \
    $(pwd)/volumes/sql/florida.gis_osm_buildings_a_free_1.sql
```


import sql into db
```sh
PGPASSWORD=secret psql \
 -h localhost -d postgres\
 -U postgres \
 -f $(pwd)/volumes/sql/florida.gis_osm_transport_free_1.sql




        gis_osm_pofw_free_1
    gis_osm_pois_free_1




```
