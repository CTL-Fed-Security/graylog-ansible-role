
# Graylog Ansible Role

## Requirements

- Ansible (> 2.5.0)
- At least 4gb of memory on the target instance.
  - Linux
    - Currently tested against:
        - Redhat / Centos / Rocky 9

To install the role, run:

    ansible-galaxy install graylog2.graylog

## Dependencies

Graylog has the following dependencies:
  - Java
  - Elasticsearch or Opensearch
  - MongoDB

See the official [Graylog documentation](https://docs.graylog.org/docs/installing) for more details on these requirements.

Be certain you are running a supported version of Elasticsearch/Opensearch. Running Graylog against an unsupported version of Elasticsearch/Opensearch can break your instance!

**Compatibility Matrix**

| Graylog version   | Minimum MongoDB Version | Maximum MongoDB Version | Minimum Opensearch Version | Maxminum Opensearch Version |
|:------------------|:-----------------------:|:-----------------------:|:--------------------------:|:---------------------------:|
| 5.2.X | 5.0.7 | 6.8 | 1.1.x | 2.13.X |
| 6.0.x | 5.0.7 | 7.x | 1.1.x (or 1.3.x for Graylog Security) | 2.15.x |
| 6.1.x | 5.0.7 | 7.x | 1.1.x (or 1.3.x for Graylog Security) | 2.15.x |


You will need to these Ansible role dependencies:
  - [Java](https://github.com/lean-delivery/ansible-role-java)

## Example Playbook

Here is an example playbook that uses this role. This is a single-instance configuration. It installs Java, MongoDB, Elasticsearch, and Graylog onto the same server.

```yaml
- hosts: "all"
  remote_user: "ubuntu"
  become: True
  vars:
    #Elasticsearch vars
    es_major_version: "7.x"
    es_version: "7.10.2"
    es_enable_xpack: False
    es_instance_name: "graylog"
    es_heap_size: "1g"
    es_config:
      node.name: "graylog"
      cluster.name: "graylog"
      http.port: 9200
      transport.tcp.port: 9300
      network.host: "127.0.0.1"
      discovery.seed_hosts: "localhost:9300"
      cluster.initial_master_nodes: "graylog"
    oss_version: True
    es_action_auto_create_index: False

    #Graylog vars
    graylog_version: 4.2
    graylog_install_java: True
    graylog_password_secret: "" # Insert your own here. Generate with: pwgen -s 96 1
    graylog_root_password_sha2: "" # Insert your own root_password_sha2 here.
    graylog_http_bind_address: "{{ ansible_default_ipv4.address }}:9000"
    graylog_http_publish_uri: "http://{{ ansible_default_ipv4.address }}:9000/"
    graylog_http_external_uri: "http://{{ ansible_default_ipv4.address }}:9000/"

  roles:
    - role: "graylog2.graylog"
      tags:
        - "graylog"
```

Remember to generate a unique `password_secret` and `root_password_sha2` for your instance.

To generate `password_secret`:

    pwgen -s 96 1

To generate `root_password_sha2`:

      echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1


## Example Playbook - Cluster

Here is an example that deploys a Graylog cluster, like the one mentioned on the [architecture page](https://docs.graylog.org/docs) of our documentation.

In our Ansible hosts file, we have 3 instances for the Elasticsearch cluster and 3 instances for the Graylog cluster:

```
[elasticsearch]
elasticsearch01
elasticsearch02
elasticsearch03

[graylog]
graylog01
graylog02
graylog03
```

First, we deploy the [Elasticsearch](https://github.com/elastic/ansible-elasticsearch) cluster. Note that this doesn't configure authentication or HTTPS. For a production instance, you would likely want that.

```yaml
- hosts: "elasticsearch"
  vars:
    es_major_version: "7.x"
    es_version: "7.10.2"
    es_enable_xpack: False
    es_instance_name: "graylog"
    es_heap_size: "1g"
    es_config:
      node.name: "{{ ansible_hostname }}"
      cluster.name: "graylog"
      http.port: 9200
      transport.port: 9300
      network.host: "0.0.0.0"
      discovery.seed_hosts: "elasticsearch01:9300, elasticsearch02:9300, elasticsearch03:9300"
      cluster.initial_master_nodes: "elasticsearch01, elasticsearch02, elasticsearch03"
    oss_version: True
    es_action_auto_create_index: False

  roles:
    - role: "elastic.elasticsearch"
```

Next, we'll deploy three MongoDB instances and configure them as a Replica Set. This is done with the [MongoDB community collection](https://github.com/ansible-collections/community.mongodb).

These MongoDB instances will live on the Graylog servers, as they are not expected to consume much resources.

Again, this doesn't configure authentication in MongoDB. You may want that for a production cluster.

```yaml
- hosts: "graylog"
  vars:
    mongodb_version: "4.4"
    bind_ip: "0.0.0.0"
    repl_set_name: "rs0"
    authorization: "disabled"
  roles:
    - community.mongodb.mongodb_repository
    - community.mongodb.mongodb_mongod
  tasks:
    - name: "Start MongoDB"
      service:
        name: "mongod"
        state: "started"
        enabled: "yes"

- hosts: "graylog01"
  tasks:
    - name: "Install PyMongo"
      apt:
        update_cache: yes
        name: "python3-pymongo"
        state: "latest"
    - name: Configure replicaset
      community.mongodb.mongodb_replicaset:
        login_host: "localhost"
        replica_set: "rs0"
        members:
        - graylog01
        - graylog02
        - graylog03
```

Finally, we install Graylog.

```yaml
- hosts: "graylog"
  vars:
    graylog_is_master: "{{ True if ansible_hostname == 'graylog01' else False }}"
    graylog_version: 4.2
    graylog_install_java: False
    graylog_install_elasticsearch: False
    graylog_install_mongodb: False
    graylog_password_secret: "" # Insert your own here. Generate with: pwgen -s 96 1
    graylog_root_password_sha2: "" # Insert your own root_password_sha2 here.
    graylog_http_bind_address: "{{ ansible_default_ipv4.address }}:9000"
    graylog_http_publish_uri: "http://{{ ansible_default_ipv4.address }}:9000/"
    graylog_http_external_uri: "http://{{ ansible_default_ipv4.address }}:9000/"
    graylog_elasticsearch_hosts: "http://elasticsearch01:9200,http://elasticsearch02:9200,http://elasticsearch03:9200"
    graylog_mongodb_uri: "mongodb://graylog01:27017,graylog02:27017,graylog03:27017/graylog"

  roles:
    - role: "graylog2.graylog"
```

We set `graylog_install_elasticsearch: False` and `graylog_install_mongodb: False` so the Graylog role doesn't try to install Elasticsearch and MongoDB. Those flags are intended for single-instance installs.

## Role Variables

A list of all available role variables is documented [here](docs/Variables.md).

## License

Apache 2.0
