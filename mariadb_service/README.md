# MariaDB/mySQL service - cslab-db

### Ansible/Heat deployment

ADAPTED FROM [Jeff Geerling's Ansible for Devops LAMP infrastructure example](https://github.com/geerlingguy/ansible-for-devops/tree/master/lamp-infrastructure).

## Description

The architecture for the MariaDB/mySQL service will be:

                    -----------------------------------
                   |  cslab-db-web (Apache/phpMyAdmin) |
                   |  Private IP + Public IP           |
                    -----------------------------------
                          /                   
     -----------------------------       ----------------------------
    |  cslab-db-0 (mySQL-Master)  | --- |  cslab-db-1 (mySQL-Slave)  |
    |  Private IP                 |     |  Private IP                |
     -----------------------------       ----------------------------

This service was designed for students taking CS 665 "Introduction to Databases" at WSU to gain hands on experience with a MariaDB/mySQL server.

The architecture offers a single MariaDB/mySQL server for users to access via a separate phpMyAdmin web server. Only the Apache/phpMyAdmin web server will be given an external IP address. If deployed within a previously built cslab private network which contains a Linux-based IDE environment, then users can access the master database server directly via mysql shell or language-specific mysql connectors from within this IDE environment.

The mySQL-Slave is set up for continuous/synchronous replication of the mySQL-Master. This slave could be configured in future for scheduled mysqldump and rsync to a backup file-server to allow for automated backup snapshots to be run without affecting user access into the master server. For database persistance, both mySQL servers are attached to Cinder virtual volumes which will not be deleted upon mySQL instance deletion.

NOTE: Currently the Apache/phpMyAdmin web server frontend is configured for insecure HTTP access on port 80. In future, additional configuration could be added to the www playbook for secure SSL/HTTPS access on port 443.

## Provisioning Notes

Only an OpenStack private cloud provisioner has been created for this MariaDB/mySQL service. However, other provisioning methods could be coded in the future.

There are three alternative OpenStack Heat template options for provisioning instances:
  1. `provisioners/stack_with_vols.yml` - This is the production-ready LAMP template which creates the architecture discussed above for cslab-db. Note: private network creation has been commented out in this template for deployment within the already running cslab private network.
  2. `provisioners/stack_no_vols.yml` - This is the same template as above except that no Cinder volumes are created and attached to the mySQL servers. All instances are purely ephemeral.
  3. `provisioners/stack_full_vamp.yml` - This template creates the web server instance(s) as a scalable resource group within the LAMP stack and adds a varnish caching HTTP proxy as the public/external frontend. If a more scalable web deployment for phpMyAdmin access was required then this template could be used, but the varnish playbook would need to be reconfigured to allow for HTTP session affinity with cookies.

## Prerequisites

Before you can run any of these playbooks, you will need to [install Ansible](http://docs.ansible.com/intro_installation.html), and run the following command to download dependencies (from within the same directory as this README file):

    $ ansible-galaxy install -r requirements.yml

For deployment on OpenStack, you will need to have all [python openstack client packages installed](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html), including the additional ["shade"](https://docs.openstack.org/shade/latest/) package. See [Ben's local_deployments openstack_client.yml file](https://github.com/benroose/eecs_ansible_local_deployments/blob/master/tasks/openstack_client.yml) for a list of required packages.

You must have also locally cloned [Ben's OpenStack Heat template modules](https://github.com/benroose/eecs_openstack_heat/tree/master/modules).

## Provision and configure the servers (OpenStack)

Pre-suppositions: You have an OpenStack private cloud available, have previously created an openrc.sh file with valid OpenStack user/project credentials, and have created an SSH public key for use within OpenStack.

To provision the virtual instances and configure them using Ansible, follow these steps (from within this directory):

  1. Add OpenStack project specific details into `provisioners/openstack.yml` and `provisioners/env_openstack_cslab.yml`
  2. Set the `provisioners/modules` soft-link to the previously cloned Heat modules directory.
  3. Adjust automatically created databases, users, and access in `playbooks/db/templates/mysql_databases.j2`
  4. Set your OpenStack credentials as environment vars: `source [path to your openrc.sh file]`
  5. Run `ansible-playbook -i inventories/openstack provision.yml`.

After everything is booted and configured, visit the public IP address of the newly created Apache/phpMyAdmin server using a web-browser and you should see the phpMyAdmin login page. The generated databases, users, and passwords will be listed in the locally generated file `/tmp/cslab_db_users`.

> If you get an error like "Failed to connect to the host via ssh: Connection timed out" or "UNREACHABLE", then check the custom ssh lamp-proxy and host configuration is correct for your network environment in `inventories/openstack/ssh.cfg` before running the `provision.yml` playbook again.

> If you get an error like "Failed to connect to the host via ssh: Host key verification failed.", then you can temporarily disable host key checking. Run the command `export ANSIBLE_HOST_KEY_CHECKING=False` and then run the `provision.yml` playbook again.

### Ancillary Notes

  - Private network IP addresses are used for all cross-instance communication (e.g. mySQL communication, MySQL master/slave replication). It is assumed the private network is secure, so port specific firewalls and mySQL privileges allow for access by any host within this private network.
  - Since a private network is used, all external Ansible connections into hosts via SSH must proxy through the public IP address of the Apache/phpMyAdmin server instance. The custom SSH proxyjump can be configured in `ansible.cfg` and `inventories/openstack/ssh.cfg`.
  - OpenStack security group `ingress web wsu net` (defined in heat templates as `sg_ingress_web`) and the `phpmyadmin_allow_access_ip` variable in `playbooks/www/vars.yml` are used to control external HTTP access into the Apache/phpMyAdmin server instance.
  - OpenStack security group `ingress ssh wsu limited net` (defined in heat templates as `sg_ingress_ssh`) is used to control external SSH/Ansible access into the Apache/phpMyAdmin server instance and for proxyjumping to the internal instances.

## About the Author

This project was modified by Ben Roose for use at Wichita State University from a base example by [Jeff Geerling](https://www.jeffgeerling.com).
