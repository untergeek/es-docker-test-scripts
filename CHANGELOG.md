# Changelog

## 1.2.0 (30 August 2024)

### Feature Release

* Add `scenarios.bash` as a source-able file to set variables based on the
  scenario. It's likely to just be a bunch of `case` statements.

* Add a `.gitignore` file

* Massive updates to `common.bash`:

  * Variables and Constants:

    * `JAVA_OPTS="-Xms512m -Xmx512m"` for overriding half of `MEMORY`, if desired.
    * `CLUSTER_NAME="docker_test"` set a new default cluster name.
    * `DISCOVERY_TYPE="single-node"` This is the default for a single node,
      but can be overriden for multi-node clusters.
    * `TRUNK=${PROJECT_NAME}-test` for making it easier to find multiple, similar nodes
    * `NUM=0` for appending a numeric value to the container name
    * `NAME=${TRUNK}-${NUM}` for naming the initial container

  * Functions

    * Add `initial_value` function to extract not only the Elasticsearch
      password, but now also the node enrollment token.
    * Updated `xpack_fork` to be quieter, to use `initial_value`, to not
      pre-emptively delete `${ENVCFG}` and `${CURLCFG}` in case another
      script sources `common.bash`.
    * Add `url_check` function to outsource repeat aliveness checks.
    * Add `start_container` function to easily add additional nodes.

* Massive updates to `create.sh`:

  * Variables and Constants:

    * `NODECOUNT=1` for the initial count of nodes to start
    * `ROLES=...` to set node roles
    * Added a second command-line argument to define a setup SCENARIO
    * Added `ES_VERSION` as an environment variable in `${ENVCFG}`

  * Execution flow changes

    * If a SCENARIO is provided at the command-line as the second argument,
      source `scenarios.bash` and have those variables imported as a result.

      * These include (so far) updating:
 
        * `NODECOUNT`
        * `DISCOVERY_TYPE`
        * `ROLES`

    * Update what is displayed during node creation
    * Call `start_container` to start nodes
    * Loop from 2 through the value of `NODECOUNT` and create new nodes with
      the values of [enrollment] `TOKEN`, `DISCOVERY_TYPE`, and `ROLES`

      * Check each node for aliveness as it comes up.

* Updates to `destroy.sh`

  * Add a `verbose` command-line option, which if specified, will show all
    containers being stopped and deleted.
  * Add a logging function to check if `verbose` was specified.

* Add `scenarios.bash`

  * Essentially a big `case` statement to assign or update variables for
    `create.sh` runs, based on the value of SCENARIO.
  * Added the `frozen_node` scenario, which sets:

    * `NODECOUNT=2`
    * `DISCOVERY_TYPE="multi-node"`
    * `ROLES="data_frozen"`

  * If a SCENARIO is specified, but does not match, print out an error to the
    command-line and exit
  
 
## 1.1.1 (24 August 2024)

### Patch Release

* Add `xpack.searchable.snapshot.shared_cache.size=50M"` to `create.sh`
  * After discovering unassigned shards in my frozen index testing, I
    discovered that `client.cluster.allocation_explain()` revealed the
    problem: 
    
    ```
    'allocate_explanation': "Elasticsearch isn't allowed to allocate this shard to any of the nodes in the cluster.
    ```

    A bit farther in, I saw:

    ```
    'explanation': 'node setting [xpack.searchable.snapshot.shared_cache.size]
    is set to zero, so shards of partially mounted indices cannot be
    allocated to this node'}]
    ```

    The solution became apparent after that.

## 1.1.0 (24 August 2024)

### Feature Release

* Add `env_var.yaml` to `docker_test` directory. This allows easy use in
  conjunction with `es_client`, which can import YAML configuration files in
  one step.

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

  It is absolutely necessary to run `source ${PROJECT_ROOT}/.env` before `Builder` can read the values in `env_var.yaml`:

## 1.0.2 (23 August 2024)

### Patch Release

* Ensure that we delete `createrepo.json` if it is successfully created

## 1.0.1 (23 August 2024)

### Patch Release

* Moved `VERSION` to `docker_test` so it's easier to see which version you have

## 1.0.0 (22 August 2024)

### Initial Release

All relevant first-release details are in `README.md`
