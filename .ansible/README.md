# Ansible Playbook
## "devops-task" scenario.

[![N|Solid](https://res.cloudinary.com/practicaldev/image/fetch/s--BkF-uzLS--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/npb90i8qo6yknbw4mpxv.png)](https://nodesource.com/products/nsolid)

Playbook for deploying node.js application to target EC2 machine running Ubuntu 20.04.3 LTS

- Containerized Postgres database 
- Nginx + Letsencrypt
- Github Action Runner

## Requirements
- Target machine (minimal 1vCPU 1G RAM and 3G disk space.)
- DNS name pointed to target machine as an ("A") Record. ex. (test.com -> 1.1.1.1)
- GitHub account for repository URL and action runner token.
- Ansible installed. (Tested  with version 2.9.6)

## Installation

1. Clone the current repository
2. Edit the "hosts" file inside of devops-task directory.
After editing hosts file you should have a total of 5 variables defined - and if done correctly it will look like this.
```sh
[targets]
123.122.121.120   ansible_connection=ssh   ansible_user=test   ansible_ssh_pass=SecretPass   ansible_sudo_pass=SecretPass  
```
After confirming you have done this part correctly, we can proceed with executing playbook.

```sh
ansible-playbook devops-task/playbook.yml -i /devops-task/hosts
```

The playbook will prompt before execution asking you to define three variables.
```sh
App name? e.g 'devops-task': 
App domain? e.g 'test.com':
GitHub Runner Token? : 
```
After defining needed variables playbook will execute roles in following order :
1. "Others"
2. "Postgres"
3. "Nginx"
4. "App"

## Others Role

Defaults:
There are 2 variables that needs to be defined : 
- github_runner_location: /opt/github-runner/
- github_repo: https://github.com/krisrazum/devops-task

Handlers:
I've created handlers for reloading following services : 
- Reload Cron
- Reload sshd

## Others - preinstall

This task is in charge of pre-installing and configuring necessary softwares and packages on the system, such as : 
- unzip 
- python3-certbot-nginx
- npm
- acl
- pm2
- setting cronjob for ssl renewal
- applying hardened sshd configuration

## Others - add-app-user
This task is in charge of adding users on the system, creating .ssh folder and pulling / appending public keys.
1. Creates one application user - using predefined variable {{ app_name }} as the user name. 
2. Creates application's user .ssh folder + touches authorized_keys file and sets the right permissions on them.
3. Creates deploy user {{ ansible_user }} .ssh folder + touches authorized_keys file and sets the right permissions on them.
4. Pulls bmijac's and krisrazum's public keys from github with CURL, registers stdout as a variable and appends both of those keys to application user and deploy user.

## Others - github-runner
This task is in charge of downloading/installing gitlab runner on the system, as well as registering it.
1. Creates directory for GitHub runner - uses a {{ github_runner_location }} variable from defaults as location.
2. Download and unarchive GitHub Runner Archive and sets directory permissions
4. Registers runner - uses {{ github_repo }} from defaults and {{ runner_token }} from user input on beginning of executing playbook.
5. Installs runner and starts runner.

## Postgres Role
I've decided to use dockerized postgres for this scenario because of seamless installation, management and in-future easily lift and shift migrations.
This approach means less overhead in scenarios such as dist upgrade trying to update postgres versions..we are safe with this one :)

Defaults:
There are 9 variables but I'd suggest not editing anything in here because theres really no reason to.
db_host is binded on local interface due to security, password for postgres users is random generated, uses a default port..
- postgres_docker_image: postgres
- postgres_docker_image_tag: "12"
- postgres_container_name: "{{ app_name }}"
- db_port: 5432
- postgres_networks: null
- container_memory_limit: null
- postgres_user: "{{ app_name }}"
- postgres_database: "{{ app_name }}"
- db_host: "127.0.0.1"

## Postgres - preinstall
This task is in charge of pre-installing and configuring necessary softwares and packages on the system for using and managing postgres, such as :
- python3-docker 
- docker.io
- libpq-dev
- python3-psycopg2

## Postgres - add-docker-container
This task will handle the password generation, containerization and management of PostgreSQL inside of container.
1. Generates password for application user and postgres (admin) user.
2. Pulls postgres image & runs docker container (binded on db_host local interface).
3. Creates application user & database.

## Nginx Role
This Role is in charge of installing software packages needed for running the webserver side, setting up vhosts + ssl, and setting up htpasswd basic authentication.

Handlers:
I've created handler for reloading following service : 
- Reload Nginx

Defaults:
There are 2 variables that needs to be defined : 
- certbot_renew_email: krisrazum@live.com
- certbot_plugin: nginx

I would recommend changing certbot_renew_email and leaving certbot_plugin as is. It is defined if anyone in future decides to use apache instead of nginx.

## Nginx - preinstall
This task is in charge of installing nginx software and enabling it as a service on boot time.
- Installs nginx
- Enables it on boot time

## Nginx - setup-vhost
This task sets up vhosts and issues letsencrypt tls certificate. 
1. Sets up a non-ssl vhost (listening on port 80 only)
2. Issue a letsencrypt certificate
3. Remove non-ssl vhost and create one vhost with both port 80 and 443 listening in one vhost. (More clear to manage this way)

## Nginx - htpasswd
This task is in charge of generating a htpasswd, encrypting it and saving the htpasswd output format to a file.
1. Generate htpasswd password
2. Encrypt htpasswd password with APR1-MD5 algorithm.
3. Copy output of htpasswd USER:PASS to /etc/nginx/.htpasswd-{{ app_name }}

## App Role
App role is in charge of deploying and installing the application.

Handlers:
I've created handlers for following stuff : 
- Execute Migrations
- Execute DBSeed
- Start Application
- Reload Nginx
- Saving the app list to be restored at reboot

Defaults:
There are 3 variables that needs to be defined : 
- npm_binary: /usr/bin/npm
- default_directory: /apps
- app_environment: development

## App - install
This task is in charge of creating application directory, cloning repositor, populating env file..will be furthermore described below.

1. Create application directory
2. Cloning a repository
3. Installing application packages from packages.json
4. Copying populated .env.{{ app_environment }} file to app
5. Sets directory permissions
6. Notify handlers


