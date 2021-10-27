

https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html
https://docs.oracle.com/en/database/oracle/oracle-database/21/comsc/installing-sample-schemas.html#GUID-3820972A-08D7-4033-9524-1E36676594EE


mkdir -p $(pwd)/volumes/zip
mkdir -p $(pwd)/volumes/sql
mkdir -p $(pwd)/volumes/bin

# login

instantclient + sqlplus + sqlldr

# download basic insta-client
curl https://download.oracle.com/otn_software/linux/instantclient/213000/instantclient-basic-linux.x64-21.3.0.0.0.zip \
    --output $(pwd)/volumes/zip/instantclient-basic-linux.x64-21.3.0.0.0.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/instantclient-basic-linux.x64-21.3.0.0.0.zip \
    -d $(pwd)/volumes/bin

# download sqlplus
curl https://download.oracle.com/otn_software/linux/instantclient/213000/instantclient-sqlplus-linux.x64-21.3.0.0.0.zip \
    --output $(pwd)/volumes/zip/instantclient-sqlplus-linux.x64-21.3.0.0.0.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/instantclient-sqlplus-linux.x64-21.3.0.0.0.zip \
    -d $(pwd)/volumes/bin



# download sqlplus
curl https://download.oracle.com/otn_software/linux/instantclient/213000/instantclient-tools-linux.x64-21.3.0.0.0.zip \
    --output $(pwd)/volumes/zip/instantclient-tools-linux.x64-21.3.0.0.0.zip

# uncompress downloaded files
unzip $(pwd)/volumes/zip/instantclient-tools-linux.x64-21.3.0.0.0.zip \
    -d $(pwd)/volumes/bin


https://download.oracle.com/otn_software/linux/instantclient/213000/instantclient-tools-linux.x64-21.3.0.0.0.zip


# instantclient_21_3
cd $(pwd)/volumes/bin/instantclient_21_3


```

export ORACLE_HOME=$(pwd)/volumes/bin/instantclient_21_3
export LD_LIBRARY_PATH="$ORACLE_HOME"
export PATH="$ORACLE_HOME:$PATH"


sqlplus -v

sqlplus system/secret123@172.18.0.1:1521/alpha

```




sqlplus system/secret123@172.18.0.1:1521/alpha
SELECT NAME, OPEN_MODE, RESTRICTED, OPEN_TIME FROM V$PDBS;

@/home/sudowing/Documents/repos/guide-local-databases/volumes/sql/db-sample-schemas-19.2/mksample secret123 secret123 secret123 secret123 secret123 secret123 secret123 secret123 USERS TEMP /tmp/oracle_demo_log 172.18.0.1:1521/alpha
