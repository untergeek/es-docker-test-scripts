# es-docker-test-scripts
Bash scripts to automate creation/destruction of Elasticsearch Docker containers for testing

The `docker_test/create.sh` script will:

  * Create a named Docker network for the container
  * Start up an Elasticsearch container running the specified version
  * Reset the password with a randomly generated one and capture it
  * Copy the generated `http_ca.crt` file from the Docker container to
    `${PROJECT_ROOT}/http_ca.crt`
  * Create a `curl` config file for easier testing
  * Wait until the node is fully running and returns a 200 HTTP response
  * Initialize the trial X-Pack license
  * Create a snapshot repository
    * The Docker container mount point (`${REPODOCKER}`) is `/media`
    * The local path is `${SCRIPTPATH}/repo`, which would usually be
      `${PROJECT_ROOT}/docker_test`
  * Export values to `${PROJECT_ROOT}/.env` for use in testing.
  * Tell you that you're ready for testing

The `docker_test/destroy.sh` script will:

  * Stop and delete the created Docker container
  * Delete the created Docker network
  * Delete the `curl` config file
  * Delete `${PROJECT_ROOT}/.env` 
  * Delete `${PROJECT_ROOT}/http_ca.crt`
  * Recursively delete the snapshot repository (`${SCRIPTPATH}/repo`)
  * Tell you the cleanup is completed
  
Environment variables used to create an Elasticsearch instance:

  * `-e "discovery.type=single-node"`
  * `-e "cluster.name=local-cluster"`
  * `-e "node.name=local-node"`
  * `-e "xpack.monitoring.templates.enabled=false"`
  * `-e "path.repo=${REPODOCKER}"`

## Preserve the `docker_test` path

The `PROJECT_NAME` is obtained automatically presuming `create.sh` is in the
`docker_test` directory when run, and that is a first-level subdirectory of the
project. If this is not the case, you may need to edit `common.bash` to set
`MANUAL_PROJECT_NAME`.

## DO NOT EDIT `ansi_clean.bash`

This file contains a function to clean ANSI control characters, including a
`CTRL-M` sequence. Changing the Elasticsearch password using the script displays
the result using ANSI colors and other control character sequences. This script
removes those so that the recovered/reset password is in plain text.

If you open `ansi_clean.bash` with an IDE or code editor, the code formatter may
try to correct those sequences and convert the actual `CTRL-M` into a string
representation, `^M`, which will prevent proper capture of the password.

If you really must know what is in there:

```
#!/bin/bash

ansi_clean () {
  # This function is separate so nobody touches the control-M sequence
  # in the second sed stream filter
  echo ${1} | sed -e 's/\x1b\[[0-9;]*m//g' -e 's/^M//g'
}
```

## `create.sh` Execution Flow

In this example, the actual path has been replaced with `${PROJECT_ROOT}`, but
the full path will be used, rather than a variable.

`${PROJECT_NAME}` is usually derived from the directory name of `${PROJECT_ROOT}`,
and is used here to show what will be rendered in real-world use.

```
cd ${PROJECT_ROOT}
docker_test/create.sh X.Y.Z

Starting container "${PROJECT_NAME}-test" from docker.elastic.co/elasticsearch/elasticsearch:X.Y.Z
Container ID: 78267407e511e5309db45840a1e9f41cf993d2e4f0340fb85566392ab2658d7e

Getting Elasticsearch credentials from container "${PROJECT_NAME}-test"...

- 22s elapsed (typically 15s - 25s)...Credentials captured!

HTTP status code for ${PROJECT_NAME}-test instance is: 200 --- ${PROJECT_NAME}-test instance is ready!

Trial license started and acknowledged. Snapshot repository "testing" created.

${PROJECT_NAME}-test container is up using image elasticsearch:X.Y.Z
Ready to test!

$PWD is ${PROJECT_ROOT}
Environment variables are in .env
```

### Contents of `.env`

In this example, the actual path has been replaced with `${PROJECT_ROOT}`, but
the full path will be used, rather than a variable.

```
cat .env
export CA_CRT=${PROJECT_ROOT}/http_ca.crt
export TEST_PATH=${PROJECT_ROOT}/docker_test
export TEST_ES_SERVER=https://127.0.0.1:9200
export TEST_ES_REPO=testing
export ESCLIENT_CA_CERTS=${PROJECT_ROOT}/http_ca.crt
export ESCLIENT_HOSTS=https://127.0.0.1:9200
export ESCLIENT_USERNAME=elastic
export TEST_USER=elastic
export ESCLIENT_PASSWORD="REDACTED"
export TEST_PASS="REDACTED"
```

### Version required

A semver styled, available version of Elasticsearch must be specified as the
single command argument. If omitted, it will yield an error:

```
cd ${PROJECT_ROOT}
docker_test/create.sh

Error! No Elasticsearch version provided.
VERSION must be in Semver format, e.g. X.Y.Z, 8.6.0
USAGE: docker_test/create.sh VERSION
```

## `destroy.sh` Execution Flow

`${PROJECT_NAME}` is usually derived from the directory name of `${PROJECT_ROOT}`,
and is used here to show what will be rendered in real-world use.

```
cd ${PROJECT_ROOT}
docker_test/destroy.sh

Stopping container ${PROJECT_NAME}-test...
${PROJECT_NAME}-test stopped.
Removing container ${PROJECT_NAME}-test...
${PROJECT_NAME}-test deleted.
Deleting remaining files and directories
Cleanup complete.
```

### Contents of `.env` (after `destroy.sh`)

`${PROJECT_ROOT}/.env` is deleted by this script:

```
cd ${PROJECT_ROOT}
cat .env
cat: .env: No such file or directory
```
