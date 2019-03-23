# docker-compose-db2-demo

## Overview

The following diagram shows the relationship of the docker containers in this docker composition.

![Image of architecture](docs/img-architecture/architecture.png)

This docker formation brings up the following docker containers:

1. *[senzing/db2express-c](https://github.com/Senzing/docker-db2express-c)*
1. *[senzing/python-demo](https://github.com/Senzing/docker-python-demo)*

Also shown in the demonstration are commands to run the following Docker images:

1. *[senzing/db2](https://github.com/Senzing/docker-db2)* in [Initialize database](#initialize-database)
1. *[senzing/g2loader](https://github.com/Senzing/docker-g2loader)* in [Run G2Loader.py](#run-g2loaderpy)
1. *[senzing/g2command](https://github.com/Senzing/docker-g2command)* in [Run G2Command.py](#run-g2commandpy)

### Contents

1. [Expectations](#expectations)
    1. [Space](#space)
    1. [Time](#time)
    1. [Background knowledge](#background-knowledge)
1. [Preparation](#preparation)
    1. [Prerequisite Software](#prerequisite-software)
    1. [Clone repository](#clone-repository)
    1. [Create SENZING_DIR](#create-senzing_dir)
1. [Using docker-compose](#using-docker-compose)
    1. [Build docker images](#build-docker-images)
    1. [Configuration](#configuration)
    1. [Launch docker formation](#launch-docker-formation)
    1. [Initialize database](#initialize-database)
    1. [Run G2Loader.py](#run-g2loaderpy)
    1. [Run G2Command.py](#run-g2commandpy)
1. [Cleanup](#cleanup)

## Expectations

### Space

This repository and demonstration require 6 GB free disk space.

### Time

Budget 2 hours to get the demonstration up-and-running, depending on CPU and network speeds.

### Background knowledge

This repository assumes a working knowledge of:

1. [Docker](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/docker.md)
1. [Docker-Compose](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/docker-compose.md)

## Preparation

### Prerequisite software

The following software programs need to be installed:

1. [docker](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-docker.md)
1. [docker-compose](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-docker-compose.md)

### Clone repository

1. Set these environment variable values:

    ```console
    export GIT_ACCOUNT=senzing
    export GIT_REPOSITORY=docker-compose-db2-demo
    ```

   Then follow steps in [clone-repository](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/clone-repository.md).

1. After the repository has been cloned, be sure the following are set:

    ```console
    export GIT_ACCOUNT_DIR=~/${GIT_ACCOUNT}.git
    export GIT_REPOSITORY_DIR="${GIT_ACCOUNT_DIR}/${GIT_REPOSITORY}"
    ```

### Create SENZING_DIR

If you do not already have an `/opt/senzing` directory on your local system, visit
[HOWTO - Create SENZING_DIR](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/create-senzing-dir.md).

## Using docker-compose

### Build docker images

1. Because an independent download is needed for the DB2 ODBC client, the
   [senzing/python-db2-base](https://github.com/Senzing/docker-python-db2-base)
   docker image must be manually built.
   Follow the build instructions at
   [github.com/Senzing/docker-python-db2-base](https://github.com/Senzing/docker-python-db2-base#build)

1. Build docker images.

    ```console
    export BASE_IMAGE=senzing/python-db2-base

    sudo docker build \
      --tag senzing/db2express-c \
      https://github.com/senzing/docker-db2express-c.git

    sudo docker build \
      --tag senzing/db2 \
      https://github.com/senzing/docker-db2.git

    sudo docker build \
      --tag senzing/python-demo \
      --build-arg BASE_IMAGE=${BASE_IMAGE} \
      https://github.com/senzing/docker-python-demo.git

    sudo docker build \
      --tag senzing/g2loader \
      --build-arg BASE_IMAGE=${BASE_IMAGE} \
      https://github.com/senzing/docker-g2loader.git

    sudo docker build \
      --tag senzing/g2command \
      --build-arg BASE_IMAGE=${BASE_IMAGE} \
      https://github.com/senzing/docker-g2command.git
    ```

### Configuration

1. **DB2_DATABASE** -
   The database schema name.
   Default: "G2"
1. **DB2_PASSWORD** -
   The password for the the database "db2inst1" user name.
   Default: "root"
1. **DB2_STORAGE** -
   Path on local system where the database files are stored.
   Default: "/storage/docker/senzing/docker-compose-db2-demo"
1. **SENZING_DIR** -
   Path on the local system where
   [Senzing_API.tgz](https://s3.amazonaws.com/public-read-access/SenzingComDownloads/Senzing_API.tgz)
   has been extracted.
   See [Create SENZING_DIR](#create-senzing_dir).
   No default.
   Usually set to "/opt/senzing".

### Launch docker formation

1. Launch docker-compose formation.  Example:

    ```console
    cd ${GIT_REPOSITORY_DIR}

    export DB2_DATABASE=G2
    export DB2_PASSWORD=db2inst1
    export DB2_STORAGE=/storage/docker/senzing/docker-compose-db2-demo
    export SENZING_DIR=/opt/senzing

    sudo docker-compose up
    ```

The database storage will be on the local system at ${db2_STORAGE}.
The default database storage path is `/storage/docker/senzing/docker-compose-db2-demo`.

### Initialize database

In a separate terminal window:

1. Determine docker network. Example:

    ```console
    sudo docker network ls

    # Choose value from NAME column of docker network ls
    export SENZING_NETWORK=nameofthe_network
    ```

1. Run `docker` command.

    ```console
    export DB2_PASSWORD=db2inst1
    export SENZING_DIR=/opt/senzing

    docker run -it  \
      --volume ${SENZING_DIR}:/opt/senzing \
      --net ${SENZING_NETWORK} \
      --env DB2INST1_PASSWORD=${DB2_PASSWORD} \
      --env LICENSE="accept" \
      senzing/db2
    ```

1. Catalog "remote" database. In docker container, run

    ```console
    su - db2inst1
    db2 catalog tcpip node G2 remote senzing-db2 server 50000
    db2 terminate
    ```

1. Create database. In docker container, run

    ```console
    db2 attach to G2 user db2inst1 using db2inst1
    db2 create database g2 using codeset utf-8 territory us
    ```

1. Populate database. In docker container, run

    ```console
    db2 connect to g2 user db2inst1 using db2inst1
    db2 -tf /opt/senzing/g2/data/g2core-schema-db2-create.sql | tee /tmp/g2schema.out
    ```

1. Exit docker container.

    ```console
    db2 connect reset
    exit
    exit
    ```

After the schema is loaded, the demonstration python/Flask app will be available at
[localhost:5000](http://localhost:5000).

### Run G2Loader.py

For more information on `senzing/g2loader` configuration and usage, see
[senzing/docker-g2loader](https://github.com/Senzing/docker-g2loader).

In a separate terminal window:

1. Determine docker network. Example:

    ```console
    sudo docker network ls

    # Choose value from NAME column of docker network ls
    export SENZING_NETWORK=nameofthe_network
    ```

1. Run `docker` command. Example:

    ```console
    export DATABASE_PROTOCOL=db2
    export DATABASE_USERNAME=db2inst1
    export DATABASE_PASSWORD=db2inst1
    export DATABASE_HOST=senzing-db2
    export DATABASE_PORT=50000
    export DATABASE_DATABASE=G2

    export SENZING_DATABASE_URL="${DATABASE_PROTOCOL}://${DATABASE_USERNAME}:${DATABASE_PASSWORD}@${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_DATABASE}"
    export SENZING_DIR=/opt/senzing

    sudo docker run \
      --env SENZING_DATABASE_URL="${SENZING_DATABASE_URL}" \
      --interactive \
      --net ${SENZING_NETWORK} \
      --rm \
      --tty \
      --volume ${SENZING_DIR}:/opt/senzing \
      senzing/g2loader \
        --purgeFirst \
        --projectFile /opt/senzing/g2/python/demo/sample/project.csv
    ```

### Run G2Command.py

For more information on `senzing/g2command` configuration and usage, see
[senzing/docker-g2command](https://github.com/Senzing/docker-g2command).

In a separate terminal window:

1. Determine docker network. Example:

    ```console
    sudo docker network ls

    # Choose value from NAME column of docker network ls
    export SENZING_NETWORK=nameofthe_network
    ```

1. Run `docker` command. Example:

    ```console
    export DATABASE_PROTOCOL=db2
    export DATABASE_USERNAME=db2inst1
    export DATABASE_PASSWORD=db2inst1
    export DATABASE_HOST=senzing-db2
    export DATABASE_PORT=50000
    export DATABASE_DATABASE=G2

    export SENZING_DATABASE_URL="${DATABASE_PROTOCOL}://${DATABASE_USERNAME}:${DATABASE_PASSWORD}@${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_DATABASE}"
    export SENZING_DIR=/opt/senzing

    sudo docker run \
      --env SENZING_DATABASE_URL="${SENZING_DATABASE_URL}" \
      --interactive \
      --net ${SENZING_NETWORK} \
      --rm \
      --tty \
      --volume ${SENZING_DIR}:/opt/senzing \
      senzing/g2command
    ```

## Cleanup

In a separate terminal window:

1. Run `docker-compose` command.

    ```console
    cd ${GIT_REPOSITORY_DIR}
    sudo docker-compose down
    ```

1. Delete database storage.

    ```console
    sudo rm -rf ${DB2_STORAGE}
    ```

1. Delete SENZING_DIR.

    ```console
    sudo rm -rf ${SENZING_DIR}
    ```

1. Delete git repository.

    ```console
    sudo rm -rf ${GIT_REPOSITORY_DIR}
    ```
