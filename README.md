# es-docker-test-scripts
Bash scripts to automate creation/destruction of Elasticsearch Docker containers
for testing.  Also included is a simple `env_var.yaml` that makes it easy to use
[`es_client`'s](https://es-client.readthedocs.io/) [`Builder`
class](https://es-client.readthedocs.io/en/latest/api.html#builder-class).

The `docker_test/create.sh` script will:

  * Create a named Docker network for the container
  * Start one or more Elasticsearch containers running the specified version
  * Capture the generated password
  * Capture the generated node enrollment token
  * Copy the generated `http_ca.crt` file from the Docker container to
    `${PROJECT_ROOT}/http_ca.crt`
  * Create a `curl` config file for easier testing
  * Wait until the node is fully running and returns a 200 HTTP response
  * Initialize the trial X-Pack license
  * Create a snapshot repository
    * The Docker container mount point (`${REPODOCKER}`) is `/media`
    * The local path is `${SCRIPTPATH}/repo`, which would usually be
      `${PROJECT_ROOT}/docker_test`
  * Export environment variables and values to `${PROJECT_ROOT}/.env` for use
    in testing. You must `source ${PROJECT_ROOT}/.env` for these to be available
    in your shell environment.
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

  * `-e "discovery.type=${DISCOVERY_TYPE}"`
  * `-e "cluster.name=${CLUSTER_NAME}"`
  * `-e "node.name=node-${VALUE}"`
  * `-e "node.roles=[${ROLES}]"`
  * `-e "xpack.searchable.snapshot.shared_cache.size=100MB"`
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

Using Elasticsearch version 8.15.0

Creating 1 node(s)...

Starting node 1...
Container ID: 70fcb95126adeac9bb1d91db148e20b8b82b7797ce1246246a1915de81a0fd25
Getting Elasticsearch credentials from container ${PROJECT_NAME}-test-0...
| 17s elapsed (typically 15s - 25s)...
Trial license started...
Snapshot repository initialized...

Node 1 started.

All nodes ready to test!

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

### Contents of `docker_test/env_var.yaml`

This is a boilerplate YAML configuration file which will work with
[`es_client`'s](https://es-client.readthedocs.io/) [`Builder`
class](https://es-client.readthedocs.io/en/latest/api.html#builder-class), once
the `.env` file has been sourced and the values assigned.

  * `env_var.yaml`:
    
    ```
    ---
    elasticsearch:
      client:
        hosts: ${ESCLIENT_HOSTS}
        ca_certs: ${ESCLIENT_CA_CERTS}
      other_settings:
        username: ${ESCLIENT_USERNAME}
        password: ${ESCLIENT_PASSWORD}
    ```
  
  * Example Flow:

    ```
    $ cd ${PROJECT_ROOT}
    $ docker_test/create.sh 8.15.0
    ### Output of create.sh here ###
    $ source .env
    $ env | grep ESCLIENT
    ESCLIENT_CA_CERTS=/path/to/http_ca.crt
    ESCLIENT_HOSTS=https://127.0.0.1:9200
    ESCLIENT_USERNAME=elastic
    ESCLIENT_PASSWORD=REDACTED
    $ python
    >>> import json
    >>> from es_client import Builder
    >>> _ = Builder(configfile='docker_test/env_var.yaml', autoconnect=True)
    >>> client = _.client
    >>> print(json.dumps(dict(client.info()), indent=2))
    {
      "name": "local-node",
      "cluster_name": "local-cluster",
      "cluster_uuid": "snJUMp3TQUWsi86EEWrU4w",
      "version": {
        "number": "8.15.0",
        "build_flavor": "default",
        "build_type": "docker",
        "build_hash": "1a77947f34deddb41af25e6f0ddb8e830159c179",
        "build_date": "2024-08-05T10:05:34.233336849Z",
        "build_snapshot": false,
        "lucene_version": "9.11.1",
        "minimum_wire_compatibility_version": "7.17.0",
        "minimum_index_compatibility_version": "7.0.0"
      },
      "tagline": "You Know, for Search"
    }
    ```

### Version required

A semver styled, available version of Elasticsearch must be specified as the
single command argument. If omitted, it will yield an error:

```
cd ${PROJECT_ROOT}
docker_test/create.sh

Error! No Elasticsearch version provided.
VERSION must be in Semver format, e.g. X.Y.Z, 8.6.0
USAGE: ./create.sh VERSION [SCENARIO]
```

### Scenario [OPTIONAL]

A second command-line option may be specified in the form of a scenario name,
which is a way to prepare one or more extra nodes with specified settings.

```
cd ${PROJECT_ROOT}
docker_test/create.sh X.Y.Z SCENARIO

Using scenario: SCENARIO
Using Elasticsearch version X.Y.Z

Creating 2 node(s)...

Starting node 1...
Container ID: 2b995646f72ca338e619c69376a888fef8a3706862318b44799908090ec722ac
Getting Elasticsearch credentials from container ${PROJECT_NAME}-test-0...
- 18s elapsed (typically 15s - 25s)...
Trial license started...
Snapshot repository initialized...

Node 1 started.
Node 2 started.

All nodes ready to test!

$PWD is ${PROJECT_ROOT}
Environment variables are in .env
```

## `destroy.sh` Execution Flow

`${PROJECT_NAME}` is usually derived from the directory name of `${PROJECT_ROOT}`,
and is used here to show what will be rendered in real-world use.

```
cd ${PROJECT_ROOT}
docker_test/destroy.sh

Stopping all containers...
Removing all containers...
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
