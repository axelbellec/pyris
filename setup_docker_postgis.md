## PostgreSQL + PostGIS setup

### Persisting Your Data

```
docker volume create pg_data
```

### Creating the Database Container

```
POSTGRES_USER=datascience
POSTGRES_PASS=datascience
POSTGRES_DBNAME=iris
POSTGRES_PORT=5452
```

### Run PostgreSQL container

```
docker run --name=postgis \
    -d -e POSTGRES_USER=$POSTGRES_USER \
    -e POSTGRES_PASS=$POSTGRES_PASS \
    -e POSTGRES_DBNAME=$POSTGRES_DBNAME \
    -e ALLOW_IP_RANGE=0.0.0.0/0 \
    -p 5432:$POSTGRES_PORT \
    -v pg_data:/var/lib/postgresql \
    --restart=always kartoza/postgis:9.6-2.4
```

#### Enter inside container

```
docker exec -it postgis /bin/bash 
```

Install some util tools
```
apt-get install unzip
```

#### Download data
```
cd home
wget https://www.data.gouv.fr/s/resources/contour-des-iris-insee-tout-en-un/20150428-161348/iris-2013-01-01.zip
unzip iris-2013-01-01.zip
```

#### Create database
```
su - postgres && cd /home
```

```
createdb pyris
psql pyris -c "CREATE EXTENSION postgis;"
echo "The database 'pyris' has been created."
```

#### Insert data
```
echo "Use 'shp2pgsql' to insert some data from the shp file"
shp2pgsql -D -W latin1 -I -s 4326 iris-2013-01-01.shp geoiris | psql -d pyris
```

```
echo "Data cleaning: remove some duplicated rows"
psql pyris -c "DELETE FROM geoiris WHERE gid IN (SELECT gid FROM (SELECT gid,RANK() OVER (PARTITION BY dcomiris ORDER BY gid) FROM geoiris) AS X WHERE X.rank > 1);"
```

```
echo "There are"
psql pyris -c "SELECT COUNT(1) FROM geoiris;"
```

#### Download infra INSEE

```
# Population 2013 - https://www.insee.fr/fr/information/2383358
# https://www.insee.fr/fr/statistiques/2386737
wget https://www.insee.fr/fr/statistiques/fichier/2386737/base-ic-evol-struct-pop-2013.zip
unzip base-ic-evol-struct-pop-2013.zip

# Logement 2013 - https://www.insee.fr/fr/information/2383355
# https://www.insee.fr/fr/statistiques/2386703
wget https://www.insee.fr/fr/statistiques/fichier/2386703/base-ic-logement-2013.zip
unzip base-ic-logement-2013.zip

# Couple, familles & ménages 2013 - https://www.insee.fr/fr/information/2383347
# https://www.insee.fr/fr/statistiques/2386664
wget https://www.insee.fr/fr/statistiques/fichier/2386664/base-ic-couples-familles-menages-2013.xls

# Diplômes, formation 2013 - https://www.insee.fr/fr/information/2383350
# https://www.insee.fr/fr/statistiques/2386698
wget https://www.insee.fr/fr/statistiques/fichier/2386698/base-ic-diplomes-formation-2013.zip
unzip base-ic-diplomes-formation-2013.zip

# Activité des résidents 2013 - https://www.insee.fr/fr/information/2383368
# https://www.insee.fr/fr/statistiques/2386631
wget https://www.insee.fr/fr/statistiques/fichier/2386631/base-ic-activite-residents-2013.zip
unzip base-ic-activite-residents-2013.zip
```

```
wget https://raw.githubusercontent.com/garaud/pyris/master/data/sql/create_infra_activites.sql
wget https://raw.githubusercontent.com/garaud/pyris/master/data/sql/create_infra_couples_familles_menages.sql
wget https://raw.githubusercontent.com/garaud/pyris/master/data/sql/create_infra_diplomes_formation.sql
wget https://raw.githubusercontent.com/garaud/pyris/master/data/sql/create_infra_logement.sql
wget https://raw.githubusercontent.com/garaud/pyris/master/data/sql/create_infra_population.sql
```

```
su - postgres && cd /home
DBNAME=pyris

psql $DBNAME -c 'CREATE SCHEMA IF NOT EXISTS insee;'

psql $DBNAME -f create_infra_activites.sql
psql $DBNAME -f create_infra_couples_familles_menages.sql
psql $DBNAME -f create_infra_diplomes_formation.sql
psql $DBNAME -f create_infra_logement.sql
psql $DBNAME -f create_infra_population.sql
```

Install `csvkit`
```
wget https://bootstrap.pypa.io/get-pip.py
python get-pip.py
pip install csvkit
```

```
ARGS="-e LATIN1 --sheet IRIS -K 5"
in2csv $ARGS base-ic-activite-residents-2013.xls > base-ic-activite-residents-2013.xls.csv
in2csv $ARGS base-ic-couples-familles-menages-2013.xls > base-ic-couples-familles-menages-2013.xls.csv
in2csv $ARGS base-ic-diplomes-formation-2013.xls > base-ic-diplomes-formation-2013.xls.csv
in2csv $ARGS base-ic-evol-struct-pop-2013.xls > base-ic-evol-struct-pop-2013.xls.csv
in2csv $ARGS base-ic-logement-2013.xls > base-ic-logement-2013.xls.csv
```

```
su - postgres && cd /home 
DBNAME=pyris

psql $DBNAME -c "COPY insee.activite FROM '/home/base-ic-activite-residents-2013.xls.csv' WITH (FORMAT CSV, HEADER TRUE);"
psql $DBNAME -c "COPY insee.formation FROM '/home/base-ic-diplomes-formation-2013.xls.csv' WITH (FORMAT CSV, HEADER TRUE);"
psql $DBNAME -c "COPY insee.logement FROM '/home/base-ic-logement-2013.xls.csv' WITH (FORMAT CSV, HEADER TRUE);"
psql $DBNAME -c "COPY insee.menage FROM '/home/base-ic-couples-familles-menages-2013.xls.csv' WITH (FORMAT CSV, HEADER TRUE);"
psql $DBNAME -c "COPY insee.population FROM '/home/base-ic-evol-struct-pop-2013.xls.csv' WITH (FORMAT CSV, HEADER TRUE);"
```

### Flask web app

#### Install dependencies
```
pip install Flask flask-restplus gunicorn psycopg2 pyaml slumber
```

Install `bower`
```
npm install -g bower
```

Download the few CSS & JavaScript dependencies
```
bower install
```

#### Launch web app

```
gunicorn -b 127.0.0.1:5555 --env PYRIS_APP_SETTINGS=./app.yml pyris.api.run:app 
```

*TODO*: `docker-compose`