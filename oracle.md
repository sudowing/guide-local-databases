

Oracle Database Enterprise Edition
Oracle Database 12c Enterprise Edition


Oracle Database Server 12c R2 is an industry leading relational database server.

The Oracle Database Server Docker Image contains the Oracle Database Server 12.2.0.1 Enterprise Edition running on Oracle Linux 7. This image contains a default database in a multi-tenant configuration with a single pluggable database.

oracle-database-enterprise-edition


step 1



https://www.oracle.com/database/technologies/oracle-database-software-downloads.html
19.3 - Enterprise Edition (also includes Standard Edition 2)
Linux x86-64	
LINUX.X64_193000_db_home.zip

# clone repo and cd into db related dir
```
git clone https://github.com/oracle/docker-images.git
cd docker-images/OracleDatabase/SingleInstance/dockerfiles
```

# Download file and place in `OracleDatabase/SingleInstance/dockerfiles/`
```
cp ~/Downloads/LINUX.X64_193000_db_home.zip 19.3.0/LINUX.X64_193000_db_home.zip
```

# build docker image
```
./buildContainerImage.sh \
    -v 19.3.0 \
    -t oracle/database:19.3.0-ee \
    -e
```

# check to see new docker image
```
docker images | grep oracle/database
oracle/database                                 19.3.0-ee       8e1e74626c10   27 seconds ago   6.67GB
```

# run docker image
```

mkdir -p $(pwd)/volumes/db/oracle

docker volume create oracle_db_data

docker run --rm -d \
    -p 1521:1521 \
    -p 5500:5500 \
    -e ORACLE_PDB=alpha \
    -e ORACLE_PWD=secret123 \
    -e ORACLE_EDITION=enterprise \
    -v oracle_db_data:/opt/oracle/oradata \
    -v $(pwd)/volumes/sql/db-sample-schemas-19.2:/sample_data \
    --name oracle_db \
    oracle/database:19.3.0-ee


docker exec -it \
    oracle_db /bin/bash

cd /sample_data

mksample secret123 secret123 secret123 secret123 secret123 secret123  secret123 secret123 EXAMPLE TEMP $ORACLE_HOME/demo/schema/log/ localhost:1521/alpha
```



# sqlplus
https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html
SQL*Plus Package (OL8 RPM)	


sudo apt-get install alien

sudo alien oracle-instantclient-sqlplus-21.3.0.0.0-1.el8.x86_64.rpm
sudo dpkg -i oracle-instantclient-sqlplus_21.3.0.0.0-2_amd64.deb


# SAMPLE DATA SCHEMA

https://docs.oracle.com/en/database/oracle/oracle-database/21/comsc/installing-sample-schemas.html#GUID-3820972A-08D7-4033-9524-1E36676594EE
















# sample schema


```sh
# create directory to hold persistant data & downloaded data
mkdir -p $(pwd)/volumes/db/oracle_db_data
mkdir -p $(pwd)/volumes/zip
mkdir -p $(pwd)/volumes/sql

# download a target data set
# https://github.com/oracle/db-sample-schemas/releases/tag/v19.2

curl https://codeload.github.com/oracle/db-sample-schemas/zip/refs/tags/v19.2 \
    --output $(pwd)/volumes/zip/db-sample-schemas-19.2.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/db-sample-schemas-19.2.zip \
    -d $(pwd)/volumes/sql

# remove compressed originals
# rm $(pwd)/volumes/zip/db-sample-schemas-19.2.zip

# replace path in template sql
cd $(pwd)/volumes/sql/db-sample-schemas-19.2
perl -p -i.bak -e 's#__SUB__CWD__#'$(pwd)'#g' *.sql */*.sql */*.dat



@/home/sudowing/Documents/repos/guide-local-databases/volumes/sql/db-sample-schemas-19.2/mksample secret123 secret123 secret123 secret123 secret123 secret123 secret123 secret123 EXAMPLE TEMP /tmp/oracle_demo_log localhost:1521/alpha


```








```

docker run --name <container name> \
    -p 1521:1521 \
    -p 5500:5500 \
    -e ORACLE_SID=<your SID> \
    -e ORACLE_PDB=alpha \
    -e ORACLE_PWD=secret123 \
    -e INIT_SGA_SIZE=<your database SGA memory in MB> \
    -e INIT_PGA_SIZE=<your database PGA memory in MB> \
    -e ORACLE_EDITION=enterprise \
    -e ORACLE_CHARACTERSET=<your character set> \
    -e ENABLE_ARCHIVELOG=true \
    -v [<host mount point>:]/opt/oracle/oradata \
    oracle/database:19.3.0-ee

```





docker run --rm \
    -p 1521:1521 \
    -p 5500:5500 \
    -e ORACLE_PDB=alpha \
    -e ORACLE_PWD=secret123 \
    -e ORACLE_EDITION=enterprise \
    --name oracle/database:19.3.0-ee \
    oracle/database:19.3.0-ee