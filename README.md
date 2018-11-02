# docker-compose-db2-demo

## Overview

This docker formation brings up the following docker containers:

1. *senzing/db2express-c*
1. *senzing/python-db2-demo*

Also shown in the demonstration are commands to run the following Docker images:

1. *senzing/db2* in [Add Senzing schemas](#add-senzing-schemas)
1. *senzing/g2loader-db2* in [Add content](#add-content)
1. *senzing/g2command-db2* in [Run G2Command.py](#run-g2commandpy)

### Contents

1. [Preparation](#preparation)
    1. [Set environment variables](#set-environment-variables)
    1. [Clone repository](#clone-repository)
    1. [Software](#software)
    1. [Docker images](#docker-images)
1. [Run Docker formation](#run-docker-formation)
    1. [Create SENZING_DIR](#create-senzing_dir)
    1. [Set environment variables](#set-environment-variables)
    1. [Launch docker formation](#launch-docker-formation)
    1. [Add Senzing schemas](#add-senzing-schemas)
    1. [Add content](#add-content)
    1. [Run G2Command.py](#run-g2commandpy)
1. [Cleanup](#cleanup)

## Preparation

### Set environment variables

These variables may be modified, but do not need to be modified.
The variables are used throughout the installation procedure.

```console
export GIT_ACCOUNT=senzing
export GIT_REPOSITORY=docker-compose-db2-demo
```

Synthesize environment variables.

```console
export GIT_ACCOUNT_DIR=~/${GIT_ACCOUNT}.git
export GIT_REPOSITORY_DIR="${GIT_ACCOUNT_DIR}/${GIT_REPOSITORY}"
export GIT_REPOSITORY_URL="https://github.com/${GIT_ACCOUNT}/${GIT_REPOSITORY}.git"
```

### Clone repository

Get repository.

```console
mkdir --parents ${GIT_ACCOUNT_DIR}
cd  ${GIT_ACCOUNT_DIR}
git clone ${GIT_REPOSITORY_URL}
```

### Software

The following software programs need to be installed.

#### docker

```console
docker --version
docker run hello-world
```

#### docker-compose

```console
docker-compose --version
```

### Docker images

Because an independent download is needed for the DB2 ODBC client, the
[senzing/python-db2-base](https://github.com/Senzing/docker-python-db2-base)
docker image must be manually built.
Follow the build instructions at
[github.com/Senzing/docker-python-db2-base](https://github.com/Senzing/docker-python-db2-base#build)

After the [senzing/python-db2-base](https://github.com/Senzing/docker-python-db2-base)
docker image is cached locally, the following docker images need to be installed.

This is the short-cut for installing all of the Senzing docker images:

```console
docker build --tag senzing/db2express-c    https://github.com/senzing/docker-db2express-c.git
docker build --tag senzing/db2             https://github.com/senzing/docker-db2.git
docker build --tag senzing/python-db2-demo https://github.com/senzing/docker-python-db2-demo.git
docker build --tag senzing/g2loader-db2    https://github.com/senzing/docker-g2loader-db2.git
docker build --tag senzing/g2command-db2   https://github.com/senzing/docker-g2command-db2.git
```

## Run Docker formation

### Create SENZING_DIR

If you do not already have an `/opt/senzing` directory on your local system, visit
[HOWTO - Create SENZING_DIR](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/create-senzing-dir.md).

### Set environment variables for docker

1. **SENZING_DIR** -
   Path on the local system where
   [Senzing_API.tgz](https://s3.amazonaws.com/public-read-access/SenzingComDownloads/Senzing_API.tgz)
   has been extracted.
   See [Create SENZING_DIR](#create-senzing_dir).
   No default.
   Usually set to "/opt/senzing".
1. **DB2INST1_PASSWORD** -
   The password for the the database "db2inst1" user name.
   Default: "root"
1. **DB2_STORAGE** -
   Path on local system where the database files are stored.
   Default: "/storage/docker/senzing/docker-compose-db2-demo"
1. Example:

    ```console
    export SENZING_DIR=/opt/senzing

    export DB2_HOST=senzing-db2
    export DB2_PORT=50000
    export DB2_STORAGE=/storage/docker/senzing/docker-compose-db2-demo
    export DB2_DATABASE=G2
    export DB2_USERNAME=db2inst1
    export DB2_PASSWORD=db2inst1
    export DB2_NETWORK=dockercomposedb2demo_backend
    ```

### Launch docker formation

```console
cd ${GIT_REPOSITORY_DIR}
docker-compose up
```

The database storage will be on the local system at ${db2_STORAGE}.
The default database storage path is `/storage/docker/senzing/docker-compose-db2-demo`.

### Add Senzing schemas

In a separate terminal window:

1. [Set environment variables for docker](#set-environment-variables-for-docker)
1. Run `docker` command.

    ```console
    docker run -it  \
      --volume ${SENZING_DIR}:/opt/senzing \
      --net ${DB2_NETWORK} \
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

### Add content

In a separate (or reusable) terminal window:

1. [Set environment variables for docker](#set-environment-variables-for-docker)
1. Run `docker` command

    ```console
    docker run -it  \
      --volume ${SENZING_DIR}:/opt/senzing \
      --net ${DB2_NETWORK} \
      --env SENZING_DATABASE_URL="db2://${DB2_USERNAME}:${DB2_PASSWORD}@${DB2_HOST}:${DB2_PORT}/${DB2_DATABASE}" \
      senzing/g2loader-db2 \
        --purgeFirst \
        --projectFile /opt/senzing/g2/python/demo/sample/project.csv
    ```

### Run G2Command.py

In a separate (or reusable) terminal window:

1. [Set environment variables for docker](#set-environment-variables-for-docker)
1. Run `docker` command

    ```console
    docker run -it  \
      --volume ${SENZING_DIR}:/opt/senzing \
      --net ${DB2_NETWORK} \
      --env SENZING_DATABASE_URL="db2://${DB2_USERNAME}:${DB2_PASSWORD}@${DB2_HOST}:${DB2_PORT}/${DB2_DATABASE}" \
      senzing/g2command-db2
    ```

## Cleanup

```console
cd ${GIT_REPOSITORY_DIR}
docker-compose down

sudo rm -rf /storage/docker/senzing/docker-compose-db2-demo
```
