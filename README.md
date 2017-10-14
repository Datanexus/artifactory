# artifactory
Playbooks/Roles used to deploy the Artifactory OSS Edition

# Installation
To install Artifactory using the [provision-artifactory.yml](provision-artifactory.yml) playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone https://github.com/Datanexus/artifactory
```

That command will pull down the repository, which includes the [provision-artifactory.yml](provision-artifactory.yml) playbook and all of its dependencies.

The only other requirements for using the playbook in this repository are a relatively recent (v2.x) release of Ansible. The easiest way to obtain a recent relese if Ansible is via a `pip install`, which requires that Python and pip are both installed locally. We have performed all of our testing using a recent (2.7.x) version of Python (Python 2); your mileage may vary if you attempt to run the playbook or the attached dynamic inventory scripts under a newer (v3.x) release of Python (Python 3).

# Using this role to deploy Artifactory
The [provision-artifactory.yml](provision-artifactory.yml) file at the top-level of this repository only supports deployments of the Artifactory OSS Edition and, as such, only supports single-node Artifactory deployments (multi-node Artifactory clusters require a license for the Artifactory Enterprise Edition). If multiple nodes are targeted by a playbook run, each of those nodes will be configured as a standalone Artifactory repository.

Since only a single-node deployment (or multiple standalone nodes) is supported by the [provision-nginx.yml](provision-nginx.yml) playbook in this repository, the usage of this playbook is quite simple when compared to many of the other DataNexus playbooks. This playbook can be used (out of the box, so to speak) to deploy a standalone Artifactory repository to an AWS environment using a command that looks something like this:

```bash
$ AWS_PROFILE=datanexus_west ./provision-artifactory.yml
```

The `AWS_PROFILE` environment variable value shown above will likely need to change to reflect the profiles configured in your local AWS credentials file.

The command used to deploy the same standalone Artifactory repository to an OSP environment is similar, but additional configuration parameters must be passed into the playbook (above and beyond those used for an AWS deployment). The easiest way to do this is to pass in a customized configuration file as an extra variable on the command-line of your Ansible playbook run:

```bash
$ ./provision-artifactory.yml -e '{ config_file: "./config-osp.yml" }'
```

here, the `./config-osp.yml` file is assumed to be a configuration file that you have created in the current working directory that contains all of the additional parameters needed for an OSP deployment. For example, you might create a file that looks something like this:

```yaml
$ cat ./config-osp.yml
---
# cloud type and VM tags
cloud: osp
tenant: datanexus
project: demo
domain: production
# configuration (internal) and accessible (external) networks
internal_uuid: 849196dc-12e9-4d50-ac52-2e753d03d13f
internal_subnet: 10.10.0.0/24
external_uuid: 92321cab-1a9c-49d6-99a1-62eedbbc2171
external_subnet: 10.10.3.0/24
# the name of the float_pool that the floating IPs assigned to
# the external subnet should be pulled from
float_pool: external
# username used to access the system via SSH
user: cloud-user
# a few configuration options related to the OpenStack
# environment we are targeting
region: region1
zone: az1
# node type and image ID to deploy; note that the image ID is optional for AWS
# deployments but mandatory for OSP deployments; if left undefined in an AWS
# deployment then then the playbook will search for an appropriate image to use
type: 'prod.tiny'
image: 'a36db534-5f9d-4a9f-bdd4-8726cc613b4e'
# the data volume size and the block pool to use when creating that data volume
data_volume: 20
block_pool: prod
# the HTTP proxy-related parameters
http_proxy: http://proxy.datanexus.corp:8080
proxy_certs_file: eg-certs.tar.gz
no_proxy: datanexus.corp
```

As you can see, there are a number of additional parameters needed for an OSP deployment (the `internal_uuid`, `external_uuid`, `float_pool`, `block_pool`, and `zone` parameters) and some of the default values from the [vars/artifactory.yml](../vars/artifactory.yml) file and the default configuration file (the [config.yml](../config.yml) file) are overridden by the values defined in this example configuration file (the `cloud`, `tenant`, `project`, `domain`, `internal_subnet`, `external_subnet`, `user`, `type`, and `region` values). There are also a few variables defined in this configuration file that are not defined in either the [vars/artifactory.yml](../vars/artifactory.yml) file or the default configuration file used when deploying to an AWS environment (the `data_volume`, `image`, `http_proxy`, `proxy_certs_file`, and `no_proxy` parameters). All of the values shown in this example file are for demonstration purposes only, and the correct values to use for these parameters will change from environment to environment. Check with the your local OSP administrator to determine the correct values to use for these parameters in your OSP environment.

## Controlling the configuration
As was mentioned briefly in the prevision section, this repository includes default values for a number of parameters in the [vars/artifactory.yml](vars/artifactory.yml) file and the default configuration file (the [config.yml](config.yml) file) that make it possible to perform deployments a standalone Artifactory repository, out of the box, with few (if any) changes necessary. If you are not happy with the default configuration defined in these two files, there are a number of ways that you can customize the configuration used for your deployment, and which method you use is entirely up to you:

* You can edit the [vars/artifactory.yml](vars/artifactory.yml) file and/or [config.yml](config.yml) file to modify the default values that are defined in those files or define additional configuration parameters
* You can pass in values as extra variables on the command-line of the Ansible playbook run to override those defined in the [vars/artifactory.yml](vars/artifactory.yml) file; values defined in the default configuration file cannot be overridden in this manner since that file is loaded last by the plays in the playbook (so values defined in a configuration file override any values defined elsewhere)
* You can setup a custom *configuration file* on the local filesystem of the Ansible host that contains the values for the parameters you wish to set or customize, then pass the location of that file into your playbook command as an extra variable

We have provided a summary of the configuration parameters that can be set (using any of these three methods) during a playbook run [here](docs/Supported-Config-Params.md). Overall, we have found the last option to be the easiest and most flexible of those three options because:

* It avoids modifying files that are being tracked under version control in the main, [artifactory repository](https://github.com/Datanexus/artifactory) (the first option); making such changes will, more than likely, lead to conflicts at a later date when these files are modified in the main [artifactory repository](https://github.com/Datanexus/artifactory) in a way that is inconsistent with the values that you have set locally (in your clone of this repository).
* It lets you maintain your preferred configuration for any given Artifactory deployment in the form of a configuration file, which you can easily maintain (along with the configuration files used for other deployments you have made) under version control in a separate repository
* It provides a record of the configuration of any given deployment, which is in direct contrast to the second option (where the configuration parameters for any given deployment are passed in on the command-line as extra variables)

That being said, the second option may be useful for some deployment scenarios (a one-off deployment of a local test environment, for example), so it remains a viable option for some users. Overall, we would recommend against trying to maintain your preferred configuration using the values defined in the [vars/artifactory.yml](vars/artifactory.yml) file or the default configuration file (the [config.yml](config.yml) file).

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions or earlier versions of these distributions (the [provision-artifactory.yml](provision-artifactory.yml)  playbook will not run successfully).
