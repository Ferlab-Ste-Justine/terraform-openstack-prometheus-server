# About

This module provision prometheus instances on openstack.

The server will fetch and continuously update its configuration files using the keys found under a given key prefix in an etcd server.

Note that after the initial fetching of the configuration files, the server does not dependend on etcd to run prometheus and is only dependent on etcd for continuously updating its configurations with future configuration changes.

# Supporting Projects

If you don't have a strategy in place to upload configuration files in etcd, consider using the following terraform resource in our etcd terraform provider: https://registry.terraform.io/providers/Ferlab-Ste-Justine/etcd/latest/docs/resources/synchronized_directory

Otherwise, the module uses the following project to update its configuration: https://github.com/Ferlab-Ste-Justine/configurations-auto-updater

# Dependencies

The server expect an etcd server running with the v3 api.

Secure communication with the etcd server (tls with either client certificate or username/password authentication) is expected also.

See the following project to setup a secure etcd cluster in openstack if you don't already have a solution: 
https://github.com/Ferlab-Ste-Justine/terraform-openstack-etcd-server
https://github.com/Ferlab-Ste-Justine/terraform-openstack-etcd-security-groups

A **prometheus.yml** configuration key is expected at the root of the keys prefix containing your configurations. You can configure other supporting configuration files (such as rules) as you wish.

# Usage

## Input

- **name**: Name of the vm
- **image_source**: Source of the image to provision the server on. It takes the following keys (only one of the two fields should be used, the other one should be empty):
  - **image_id**: Id of the image to associate with a vm that has local storage
  - **volume_id**: Id of a volume containing the os to associate with the vm
- **data_volume_id**: Id for an optional separate volume to attach to the vm on prometheus' data path
- **flavor_id**: Id of the vm flavor to assign to the instance. See hardware recommendations to make an informed choice: https://etcd.io/docs/v3.4/op-guide/hardware/
- **network_port**: Resource of type **openstack_networking_port_v2** to assign to the vm for network connectivity
- **server_group**: Server group to assign to the node. Should be of type **openstack_compute_servergroup_v2**.
- **keypair_name**: Name of the ssh keypair that will be used to ssh against the vm.
- **etcd**: Parameters to connect to the etcd backend. It has the following keys:
  - **ca_certificate**: Tls ca certificate that will be used to validate the authenticity of the etcd cluster
  - **etcd_key_prefix**: Prefix for all the domain keys. The server will look for keys with this prefix and will remove this prefix from the key's name to get the domain.
  - **etcd_endpoints**: A list of endpoints for the etcd servers, each entry taking the ```<ip>:<port>``` format
  - **client**: Authentication parameters for the client (either certificate or username/password authentication are support). It has the following keys:
    - **certificate**: Client certificate if certificate authentication is used.
    - **key**: Client key if certificate authentication is used.
    - **username**: Client username if certificate authentication is used.
    - **password**: Client password if certificate authentication is used.
- **prometheus**: Parameters to customise the behavior of prometheus. It has the following keys:
  - **web**: Object containing the following keys:
    - **external_url**: Value for the **--web.external-url** prometheus command line parameter. Has to be defined.
    - **max_connections**: Value for the **--web.max-connections** prometheus command line parameter. Set to 0 to use the default value (512).
    - **read_timeout**: Value for the **--web.read-timeout** prometheus command line parameter. Set to the empty string to use the default value (5m).
  - **retention**: Object containing the following keys:
    - **time**: Value for the **--storage.tsdb.retention.time** prometheus command line parameter. Set to the empty string to use the default value (15d).
    - **size**: Value for the **--storage.tsdb.retention.size** prometheus command line parameter. Set to the empty string to use the default value (0 for unlimited size).
- **chrony**: Optional chrony configuration for when you need a more fine-grained ntp setup on your vm. It is an object with the following fields:
  - **enabled**: If set the false (the default), chrony will not be installed and the vm ntp settings will be left to default.
  - **servers**: List of ntp servers to sync from with each entry containing two properties, **url** and **options** (see: https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#server)
  - **pools**: A list of ntp server pools to sync from with each entry containing two properties, **url** and **options** (see: https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#pool)
  - **makestep**: An object containing remedial instructions if the clock of the vm is significantly out of sync at startup. It is an object containing two properties, **threshold** and **limit** (see: https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#makestep)
- **fluentbit**: Optional fluent-bit configuration to securely route logs to a fluend/fluent-bit node using the forward plugin. Alternatively, configuration can be 100% dynamic by specifying the parameters of an etcd store or git repo to fetch the configuration from. It has the following keys:
  - **enabled**: If set the false (the default), fluent-bit will not be installed.
  - **prometheus_tag**: Tag to assign to logs coming from the prometheus process.
  - **prometheus_updater_tag**: Tag to assign to logs coming from the process that updates the prometheus configurations.
  - **node_exporter_tag** Tag to assign to logs coming from the prometheus node exporter
  - **forward**: Configuration for the forward plugin that will talk to the external fluend/fluent-bit node. It has the following keys:
    - **domain**: Ip or domain name of the remote fluend node.
    - **port**: Port the remote fluend node listens on
    - **hostname**: Unique hostname identifier for the vm
    - **shared_key**: Secret shared key with the remote fluentd node to authentify the client
    - **ca_cert**: CA certificate that signed the remote fluentd node's server certificate (used to authentify it)
- **fluentbit_dynamic_config**: Optional configuration to update fluent-bit configuration dynamically either from an etcd key prefix or a path in a git repo.
  - **enabled**: Boolean flag to indicate whether dynamic configuration is enabled at all. If set to true, configurations will be set dynamically. The default configurations can still be referenced as needed by the dynamic configuration. They are at the following paths:
    - **Global Service Configs**: /etc/fluent-bit-customization/default-config/service.conf
    - **Default Variables**: /etc/fluent-bit-customization/default-config/default-variables.conf
    - **Systemd Inputs**: /etc/fluent-bit-customization/default-config/inputs.conf
    - **Forward Output For All Inputs**: /etc/fluent-bit-customization/default-config/output-all.conf
    - **Forward Output For Default Inputs Only**: /etc/fluent-bit-customization/default-config/output-default-sources.conf
  - **source**: Indicates the source of the dynamic config. Can be either **etcd** or **git**.
  - **etcd**: Parameters to fetch fluent-bit configurations dynamically from an etcd cluster. It has the following keys:
    - **key_prefix**: Etcd key prefix to search for fluent-bit configuration
    - **endpoints**: Endpoints of the etcd cluster. Endpoints should have the format `<ip>:<port>`
    - **ca_certificate**: CA certificate against which the server certificates of the etcd cluster will be verified for authenticity
    - **client**: Client authentication. It takes the following keys:
      - **certificate**: Client tls certificate to authentify with. To be used for certificate authentication.
      - **key**: Client private tls key to authentify with. To be used for certificate authentication.
      - **username**: Client's username. To be used for username/password authentication.
      - **password**: Client's password. To be used for username/password authentication.
  - **git**: Parameters to fetch fluent-bit configurations dynamically from an git repo. It has the following keys:
    - **repo**: Url of the git repository. It should have the ssh format.
    - **ref**: Git reference (usually branch) to checkout in the repository
    - **path**: Path to sync from in the git repository. If the empty string is passed, syncing will happen from the root of the repository.
    - **trusted_gpg_keys**: List of trusted gpp keys to verify the signature of the top commit. If an empty list is passed, the commit signature will not be verified.
    - **auth**: Authentication to the git server. It should have the following keys:
      - **client_ssh_key** Private client ssh key to authentication to the server.
      - **server_ssh_fingerprint**: Public ssh fingerprint of the server that will be used to authentify it.
- **prometheus_secrets**: List of prometheus secrets (to access exporters, alertmanagers and other sattelite processes) to pass to the server's filesystem. The prometheus user that the prometheus process runs as will be made owner and given exclusive access to these files. Each element in the list takes the following keys:
  - **path**: Filesystem path where to store the secret on the server
  - **content**: Value of the secret
- **install_dependencies**: Whether cloud-init should install external dependencies (should be set to false if you already provide an image with the external dependencies built-in).