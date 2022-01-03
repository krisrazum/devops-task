# Ansible Playbook
## "devops-task" scenario by ArsFutura.

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

## Others

This role is in charge of pre-installing and configuring necessary softwares and packages on the system, such as : 
- unzip 
- python3-certbot-nginx
- npm
- acl
- pm2
- setting cronjob for ssl renewal
