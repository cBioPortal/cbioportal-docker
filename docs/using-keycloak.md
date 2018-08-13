# Authenticate using Keycloak #

This guide describes a way to Dockerise Keycloak along with
cBioPortal, for authentication as described in
the
[cBioPortal documentation](https://cbioportal.readthedocs.io/en/latest/Authenticating-and-Authorizing-Users-via-keycloak.html#introduction).

First, create an isolated network in which the Keycloak and MySQL
servers can talk to one another.

```shell
docker network create kcnet
```

Run a MySQL database in which Keycloak can store its data. This
database server will not be addressable from outside the Docker
network. The database will store its files in a folder named
`kcdb-files` in the present working directory, unless you specify some
other (absolute) path before the colon in the `-v` argument.

```shell
docker run -d --restart=always \
    --name=kcdb \
    --net=kcnet \
    -v "$PWD/kcdb-files:/var/lib/mysql" \
    -e MYSQL_DATABASE=keycloak \
    -e MYSQL_USER=keycloak \
    -e MYSQL_PASSWORD=password \
    -e MYSQL_ROOT_PASSWORD=root_password \
    mysql
```

Then run the actual Keycloak server, using
[this image](https://hub.docker.com/r/jboss/keycloak/)
available from Docker Hub. This will by default connect to the
database using the (non-root) credentials in the example above. The
server will be accessible to the outside world on port 8180, so make
sure to choose a strong administrator password.

The command below uses the default values for `MYSQL_DATABASE`, `MYSQL_USER` and `MYSQL_PASSWORD` (listed in the command above). If you wish to change these credentials, specify them in the command below. For instance, if `MYSQL_USER` in the database container is `user`, you need to add `-e MYSQL_USER=user`.

```
docker run -d --restart=always \
    --name=cbiokc \
    --net=kcnet \
    -p 8180:8080 \
    -e MYSQL_PORT_3306_TCP_ADDR=kcdb \
    -e MYSQL_PORT_3306_TCP_PORT=3306 \
    -e KEYCLOAK_USER=admin \
    -e "KEYCLOAK_PASSWORD=<admin_password_here>" \
    -e DB_VENDOR="MYSQL" \
    jboss/keycloak
```

Finally, configure Keycloak and cBioPortal as explained in the
[cBioPortal documentation](https://cbioportal.readthedocs.io/en/latest/Authenticating-and-Authorizing-Users-via-keycloak.html#configure-keycloak-to-authenticate-your-cbioportal-instance).
Click [here](adjusting_portal.properties_configuration.md) for an
explanation on how to adjust portal properties used when building a
Docker image for cBioPortal, and remember to specify port 8180 for the
Keycloak server, wherever the guide says 8080.
