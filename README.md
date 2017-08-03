# cbioportal-docker @ The Hyve

The [cBioPortal](https://github.com/cBioPortal/cbioportal) project documents a setup to deploy a cBioPortal server using Docker, in [this section of the documentation](https://cbioportal.readthedocs.io/en/latest/#docker). As cBioPortal traditionally does not distinguish between build-time and deploy-time configuration, the setup documented there builds the application at runtime, and suggests running auxiliary commands in the same container as the webserver. The above approach may sacrifice a few advantages of using Docker by going against some of its idioms. For this reason, the project you are currently looking at documents an alternative setup, which builds a ready-to-run cBioPortal application into a Docker image.

To get started, download and install Docker from www.docker.com.

[Notes for non-Linux systems](notes-for-non-linux.md)

## Usage instructions ##

### Step 1 - Setup network ###
Create a network in order for the cBioPortal container and mysql database to communicate.
```
docker network create cbio-net
```

### Step 2 - Run mysql with seed database ###
Download the seed database from [cBioPortal Datahub]( https://github.com/cBioPortal/datahub/blob/bee2a285d4c93cd658b5af30ace6fc33192d8190/seedDB/README.md).

:warning: Make sure to replace `/<path_to_seed_database>/cbioportal-seed_<genome_build>_<seed_version>` with the path and name of the downloaded seed database. The command below imports the seed database files into a MySQL database stored in `/<path_to_save_mysql_db>/db_files/`. These should be absolute paths.

:information_source: The protein database (PDB) data is relatively large and can therefore take 45 minutes to load. You can skip loading this part of the seed database by removing the line that loads `cbioportal-seed_<genome_build>_<seed_version>_only-pdb.sql.gz`. Please note that your instance will be missing the 3D structure view feature (in the mutations view) if you chose to leave this out.

```
docker run -d --restart=always \
  --name='cbioDB' \
  --net=cbio-net \
  -e MYSQL_ROOT_PASSWORD=P@ssword1 \
  -e MYSQL_USER=cbio \
  -e MYSQL_PASSWORD=P@ssword1 \
  -e MYSQL_DATABASE=cbioportal \
  -v /<path_to_save_mysql_db>/db_files/:/var/lib/mysql/ \
  -v /<path_to_seed_database>/cgds.sql:/docker-entrypoint-initdb.d/cgds.sql:ro \
  -v /<path_to_seed_database>/seed-cbioportal_<genome_build>_<seed_version>.sql.gz:/docker-entrypoint-initdb.d/seed_part1.sql.gz:ro \
  -v /<path_to_seed_database>/seed-cbioportal_<genome_build>_<seed_version>_only-pdb.sql.gz:/docker-entrypoint-initdb.d/seed_part2.sql.gz:ro \
  mysql
```

Make sure to follow the logs of this step to ensure no errors occur. Run this command:
```
docker logs -f cbioDB
```
If any error occurs, make sure to check it. A common cause is pointing the `-v` parameters above to folders or files that do not exist.

### Step 3 - Build the Docker image containing cBioPortal ###
Checkout the repository, enter the directory and run build the image.

```
git clone https://github.com/thehyve/cbioportal-docker.git
cd cbioportal-docker
docker build -t cbioportal-image .
```

Alternatively, if you do not wish to change anything in the Dockerfile or the properties, you can run:

```
docker build -t cbioportal-image https://github.com/thehyve/cbioportal-docker.git
```

If you want to change any variable defined in portal.properties,
have a look [here](adjusting_portal.properties_configuration.md).
If you want to build an image based on a different branch, you can
read [this](adjusting_Dockerfile_configuration.md).

### Step 4 - Update the database schema ###
Update the seeded database schema to match the cBioPortal version
in the image, by running the following command. Note that this will
most likely make your database irreversibly incompatible with older
versions of the portal code.

```
docker run --rm -it --net cbio-net \
    cbioportal-image \
    migrate_db.py -p /cbioportal/src/main/resources/portal.properties -s /cbioportal/db-scripts/src/main/resources/migration.sql
```

### Step 5 - Run the cBioPortal web server ###
```
docker run -d --restart=always \
    --name=cbioportal-container \
    --net=cbio-net \
    -e CATALINA_OPTS='-Xms2g -Xmx4g' \
    -p 8081:8080 \
    cbioportal-image
```

On server systems that can easily spare 4 GiB or more of memory,
set the `-Xms` and `-Xmx` options to the same number. This should
increase performance of certain memory-intensive web services such
as computing the data for the co-expression tab. If you are using
MacOS or Windows, make sure to take a look at [these
notes](notes-for-non-linux.md) to allocate more memory for the
virtual machine in which all Docker processes are running.

cBioPortal can now be reached at http://localhost:8081/cbioportal/

Activity of Docker containers can be seen with:
```
docker ps -a
```

## Data loading & more commands ##

For more uses of the cBioPortal image, see [example_commands.md](example_commands.md)

## Uninstalling cBioPortal ##
First we stop the Docker containers.
```
docker stop cbioDB
docker stop cbioportal-container
```

Then we remove the Docker containers.
```
docker rm cbioDB
docker rm cbioportal-container
```

Cached Docker images can be seen with:
```
docker images
```

Finally we remove the cached Docker images.
```
docker rmi mysql
docker rmi cbioportal-image
```
