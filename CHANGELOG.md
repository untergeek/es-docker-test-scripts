# Changelog

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
