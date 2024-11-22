# Ansible Playbooks for Decidim application on RHEL / CentOS / Rocky Linux

> **Operating system main release stream : 9**

These playbooks have been tested with the following versions : 
- Rocky Linux 9.4 (Blue Onyx)
- Ansible 2.18.0
- Python 3.11.4 (operator server) / 3.9.18 (target server) 
- PostgreSQL 13.16
- NGINX 1.24.0
- Redis 6.2.7
- Node JS 16.20.2
- ruby 3.0.6


## Glossary

**Operator** :  
This is the server / machine that will be executing these Ansible playbooks. This could be your local machine.

**Target** :  
This is the server on which the playbooks will install manage the Decidim application.

## Requirements 

### Operator machine

- Install Python3  
https://realpython.com/installing-python/
- Install Ansible  
https://docs.ansible.com/ansible/latest/installation_guide/index.html
- Install the Ansible PostgreSQL community module
https://galaxy.ansible.com/ui/repo/published/community/postgresql/
```shell
ansible-galaxy collection install community.postgresql
```
- Check that the operator can connect via SSH to the target server. SSH keys are encouraged but you can update the ansible playbook commands for password prompt (`--ask-pass` options)  
https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html
- Clone this repository to get the playbooks  
```shell
git clone https://github.com/OpenSourcePolitics/ansible-decidim-rhel
cd ansible-decidim-rhel
```

### Target server

- Can be reached via SSH with keys or password
- Have a user with sudo right without password
- We strongly recommend to make a backup image of your server to be able to restart the installtion from zero if needed

## Ansible inventory configuration

By default, Ansible will use a file named `inventory.yml` to find configuration variables used by the playbooks.  
This file is not present in this repository beacause it can contain sensible informations about your target server and API keys.  

You need to copy the `inventory-example.yml` file and change the variables to suit your configuration.

```shell
cp inventory-example.yml inventory.yml
```

> Note : `{{ ansible_user }}` is automatically replaced by the target user used by Ansible

| Variable | Example | Description |
| -------- | ------- | ----------- |
| `ansible_python_interpreter` | `/usr/bin/python3` | path to the python interpreter on the target server |
| `ansible_host` | `10.10.10.10` | SSH : host or IP address of the target server |
| `ansible_port` | `22` | SSH : port used to connect to the target server |
| `git_root` | `https://github.com` | Root url of the Git repository hosting the Decidim code being deploy |
| `git_organization` | `OpenSourcePolitics` | Git organization of the Git repository |
| `git_repository` | `decidim-app` | Name of the Git repository |
| `repository_branch` | `master` | Branch or tag of the Git repository |
| `decidim_home_path` | `/home/{{ ansible_user }}` | Home directory of the target user using the Decidim code being deploy |
| `decidim_deployment_path` | `/home/{{ ansible_user }}` or `/var/www` | Root path where the code is deployed |
| `app_repository` | `decidim-app` | Directory name where the code is deployed (your code will be deployed to `{{ decidim_deployment_path }}/{{ app_repository }}`) |
| `decidim_host` | `decidim.target.com` | Hostname on which your Decidim platform will be available. Used by NGINX for web request routing and Certbot / Let's Encrypt SSL certificate |
| `certbot_email_address` | `admin-sys@your-company.com` | Email address used for Certbot / Let's Encrypt SSL certificate issuing |
| `database_name` | `decidim_db` | Name of the Postgres database used by Decidim |
| `secret_key_base` | `xxxxxx` | Decidim ENV variable : Secret key base for the Application. It is specially important that this is kept secret from the outside word.  |
| `default_locale` | `en` | Decidim ENV variable : default locale for the platform |
| `available_locales` | `en,fr,ca,es` | Decidim ENV variable : available locales for the platform |
| `geocoder_lookup_api_key` | `xxxxxx` | Decidim ENV variable : Here.con API key for Geocoding addresses |
| `object_storage_access_key` | `xxxxxx` | Decidim ENV variable : Object Storage (S3, Minio etc.) access key (used for the platform migration) |
| `object_storage_secret_key` | `xxxxxx` | Decidim ENV variable : Object Storage (S3, Minio etc.) secret key (used for the platform migration) |
| `object_storage_bucket_name` | `decidim-bucket` | Decidim ENV variable : Object Storage (S3, Minio etc.) bucket name (used for the platform migration) |
| `object_storage_endpoint_host` | `decidim.storage.opensourcepolitics.eu` | Decidim ENV variable : Object Storage (S3, Minio etc.) endpoint (used for the platform migration) |
| `decidim_admin_email` | `administrator@email.com` | Email address of one Decidim administrator (used for the platform migration) |
| `postgres_restore_file_path` | `/home/{{ ansible_user }}/postgres-database-dump.sql` | Absolute path of the database dump to be imported |

## Ansible Playbooks

> **Note :**  
> - replace `< ansible_user >` with the user configured on the target server
> - replace `< ssh_key_path >` with the path to the SSH key used to connect to the target server
> - you can add the `-i my-custom-inventory.yml` if your Ansible inventory is not in the default `inventory.yml` file

### `playbook-install.yml` : Decidim installation
```shell
ansible-playbook -u < ansible_user > --private-key=< ssh_key_path > playbook-install.yml
```

This playbook will : 
- install all the dependencies needed by the Decidim runtime
- install and configure NGINX with a SSL certificate (Let's Encrypt)
- install and configure PostgreSQL and create an empty database with all the tables needed for Decidim
- install and configure Decidim 
- Start all the services needed to run the platform

At the end of this playbook, your Decidim platform will be accessible at `https://<decidim_host>/`  
(`decidim_host` is the variable set in your Ansible inventory)


### `playbook-bump.yml` : Decidim basic update
```shell
ansible-playbook -u < ansible_user > --private-key=< ssh_key_path > playbook-bump.yml
```

This playbook will : 
- update the Decidim code the latest version of the git configuration from your inventory (repository, branch etc.)
- install / update the ruby decidim dependencies
- run the database migrations if needed
- Restart all the services needed to run the platform


### `playbook-import-database.yml` : Import exiting Postgres database
```shell
ansible-playbook -u < ansible_user > --private-key=< ssh_key_path > playbook-import-database.yml
```

> Note: 
> - Your need to set up the inventory variable `postgres_restore_file_path` with the path to a database dump located on your target server
> - `< database_name >` is the name of the database that will be used for backup, creation and import

This playbook will : 
- backup the `< database_name >` database if it exists
- drop the `< database_name >` database if it exists
- (re)create the `< database_name >` database 
- import `< postgres_restore_file_path >` into the `< database_name >` database
- run the database migrations if needed
- Restart all the services needed to run the platform


### `playbook-post-upgrade-0.27.yml` : Run additionnal scripts when upgrading to Decidim 0.27.x version
```shell
ansible-playbook -u < ansible_user > --private-key=< ssh_key_path > playbook-post-upgrade-0.27.yml
```

This playbook needs to be run if you import or use a database that was running Dedidim 0.26.x and you upgrade to 0.27

This playbook will run all the additionnal scripts mentionned in the [Decidim 0.27 update Change Log](https://github.com/decidim/decidim/blob/release/0.27-stable/CHANGELOG.md)


## Useful commands

