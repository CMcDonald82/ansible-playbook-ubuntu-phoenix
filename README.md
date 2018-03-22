# ansible-playbook-ubuntu-phoenix
Ansible playbook for setting up an Ubuntu server to run a Phoenix (Elixir) app

This playbook will install and configure the following 

## Requirements

### Local Machine
* Install [Ansible](http://docs.ansible.com/ansible/latest/intro_installation.html)

* Install the roles listed in install_roles.yml
```
ansible-galaxy install -r install_roles.yml
```

### Target Machine (Server)
* Server running Ubuntu 16.04 LTS (Xenial Xerus). You will also need a domain name that you can point to this server.

## Dependencies
Since these are Python packages, I recommend creating a [virtualenv](https://virtualenv.pypa.io/en/stable/) to install these packages in to keep them isolated from the rest of your local environment. I personally use [pyenv](https://github.com/pyenv/pyenv) to manage Python versions locally (I highly recommend this if you're developing on a Mac) and [pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv) for managing virtualenvs.

Create and activate a virtualenv locally (using Python > 3.3), then install the following packages: 

* [Ansible](http://docs.ansible.com/ansible/latest/intro_installation.html#latest-releases-via-pip)
```
pip install ansible
```
* [Passlib](https://passlib.readthedocs.io/en/stable/install.html#supported-platforms) (for generating strong passwords via the command line)
```
pip install passlib 
```

## Setup

1. Create a server (on an IaaS provider such as DigitalOcean, AWS, etc or even a physical machine you own). Make sure the machine is running Ubuntu 16.04. If you already have a server you wish to provision using this project, ignore this step.

2. Specify the IP address of the server you wish to deploy to in the hosts file (environments/production/hosts).

NOTE: Feel free to create more environments in the /environments dir - you can name them whatever you want and create separate group_vars & host_vars directories for each one. When adding more environments, be sure to create a hosts file for each one so you can specify the IP address of the server it will be deploying to.

3. Obtain a domain name for your app (this can either be purchased through a Domain Name Registrar such as NameCheap or setup for free with a service such as Freenom).

4. Setup DNS (Nameservers, A-Records, etc.) for your domain name. A good tutorial on this is [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-host-name-with-digitalocean). Although this tutorial is specific to DigitalOcean, the information and steps are generally applicable if hosting on another service such as AWS.

## A Note On Using Ansible Vault
NOTE: you can change the editor that is used to open files when running Ansible Vault commands 
See [this article](https://www.digitalocean.com/community/tutorials/how-to-use-vault-to-protect-sensitive-ansible-data-on-ubuntu-16-04)

ex.) To change the default editor to Sublime Text:
```
export EDITOR='subl -w'
```

* Create a file called .vault_password.txt within the top-level directory. This file will contain a single line which is your private vault password for encrypting/decrypting secrets. I recommend generating a strong one with the following command using the passlib library:
```
python -c "from passlib.hash import sha512_crypt; import getpass; print sha512_crypt.using(rounds=5000).hash(getpass.getpass())"
```
The .vault_password.txt file is ignored in the .gitignore file so it will not be committed to the repo. This helps prevent accidentally distributing sensitive information.

## Usage
* Pass variables to roles. Generally, there are 3 ways that seem to work best:
```  
  1. Place the variables in files in group_vars/all directory (place non-sensitive variables in vars.yml, place sensitive variables such as passwords and secrets in vault.yml (encrypted)). I usually create a separate subdirectory for each role under the group_vars/all directory to keep the variables for each role separate. Each of these role-specific subdirectories contains a vars.yml and vault.yml file. This is the setup used in this project, so just look at the directory structure for an example.

  2. Use the vars or vars_files entries in the playbook:
  
  ex.)
    - hosts: all
      vars:
        variable1: value1
        variable2: value2
        etc...
      vars_files:
        - path/to/vars_file1
        - path/to/vars_file2
        etc...

  3. Specify the variables directly with the role:
  
  ex.)  
    roles:
      - role: role_name
        variable1: value1
        variable2: value2
        etc...
  
```
This playbook is setup using the 1st way of passing variables to roles. A list of all the variables necessary for the roles can be found in the Variables section below. Make sure all the variables in all the files in the directories under 
environments/production/group_vars/all have been properly set.

* Encrypt all the 'vault.yml' files in the various directories under environments/production/group_vars/all
```
ansible-vault encrypt environments/production/group_vars/all/<role_name>/vault.yml
```

* Encrypt the serverblock files
```
ansible-vault encrypt templates/nginx/serverblocks/<name_of_serverblock>.conf
```

* Run the playbook
```
ansible-playbook site.yml
```


## Variables

* Global
vars.yml:
```
remote_username: "{{ vault_deploy_username }}"
```
This is the username that is used when running commands via Ansible on the remote server. Since Ansible will SSH into the remote server as this user, it is important that the user specified here has permission to SSH into the remote server. The two most common values for this variable will be 'root' (used the first time the server is provisioned, since root will be the only user that exists on the server at that point) and the user that is created as part of this playbook for the purpose of deploying the app (usually named 'deploy'). Once the server has been provisioned, the 'deploy' user will exist on the server and the 'root' user will not be able to SSH in anymore so we will need to make sure this variable is set to the 'deploy' user on subsequent runs of the playbook.

```
phoenix_otp_app_name: new_app
```
This is the OTP-style, "snake_case" format of the name of the app you're deploying to the server. This variable is needed mainly for 2 reasons: to be set as an environment variable, which will be used by the vm.args file that Erlang uses to configure the app, and in the custom Nginx serverblock file as part of the path to the location where the static assets will be served from once the app is built and deployed as an Elixir release to the server.   

* Users
```
# vars.yml

users:
  - username: "{{ vault_deploy_username }}"
    password: "{{ vault_deploy_password }}"
    groups:
      - sudo
    ssh_keys:
      - ~/.ssh/id_rsa.pub
```
This variable is a list of users that will be created on the Ubuntu system by the Users role. Only one user is necessary (the 'deploy' user) but you can create as many as you'd like. See the [repo](https://github.com/CMcDonald82/ansible-role-users) for more details.
```
# vault.yml

vault_deploy_username: deploy
vault_deploy_password: vault_deploy_password # Replace this with a password generated using passlib
```
These are the username and password, respectively, of the main 'deploy' user that will be created on the Ubuntu system for deploying the app. Store these in a vault-encrypted file for security. Generate the vault_deploy_password using the same passlib script from the "A Note On Using Ansible Vault" section above (just make sure to run it again to create a new password separate from the one used in .vault_password.txt)

* Postgresql
```
# vars.yml

postgresql_databases: 
  - name: "{{ vault_postgresql_db_name }}"

postgresql_users: 
  - name: "{{ vault_deploy_username }}"
    password: "{{ vault_postgresql_db_password }}" 
    encrypted: yes
    db: "{{ vault_postgresql_db_name }}"
    role_attr_flags: "SUPERUSER,LOGIN"

postgresql_hba_entries: 
  - type: local
    user: "{{ vault_deploy_username }}"
    method: md5
    database: all 
```
These are the databases and users (Postgresql users, not Ubuntu users) that will be created by the Postgresql role. This is where you'd specify the params to create the database you want to use with your Phoenix app that you'll be deploying. Make sure the 'db' field for each user in postgresql_users matches a db that is being created in postgresql_databases (or a PostgreSQL database that has already been created on the target server).
```
# vault.yml

vault_postgresql_db_name: "phoenix_db" # Replace this with whatever you want to name the production database
vault_postgresql_db_username: "{{ vault_deploy_username }}"
vault_postgresql_db_password: "postgres" # Replace this with a more secure password of your choice
vault_postgresql_db_hostname: "localhost"
```
These are (obviously) the database parameters used to create and connect to the Postgres database. These variables in turn can be used to set the environment variables that will be read by the Phoenix app as well as in the psotgresql vars above (such as postgresql_databases and postgresql_users).

* SSL
```
# vars.yml

ssl_create_cert: true
ssl_email: "{{ vault_ssl_email }}"
ssl_key_path: "/etc/letsencrypt/live/{{ ssl_domain_name }}/privkey.pem"
ssl_cert_path: "/etc/letsencrypt/live/{{ ssl_domain_name }}/fullchain.pem"
ssl_domain_name: "{{ vault_ssl_domain_name }}"
ssl_hostnames: 
  - "{{ ssl_domain_name }}"
  - "www.{{ ssl_domain_name }}"
```
These can basically be left as-is since they just use the vault_ssl_* vars for the most part.
```
# vault.yml

vault_ssl_email: myemail@example.com # Replace with a real, valid email address that you own
vault_ssl_domain_name: example.com # Replace with the domain name you obtained and setup for this server
```
These are the email and domain name that will be used by Certbot to generate and renew certs.

* Nginx
```
# vars.yml

nginx_sites_available_files: 
  - ./templates/nginx/serverblocks/phoenix-example.conf
```
This is the path to the custom serverblock file(s) for your phoenix app. You can create multiple such files, just put the path to each one here. This way you can tailor your Nginx configurations based on the architecture of the specific app you're deploying while reusing this playbook across different projects.

* Environment
```
# vars.yml

environment_vars:
  REPLACE_OS_VARS: true
  DB_NAME: "{{ vault_postgresql_db_name }}"
  DB_USERNAME: "{{ vault_postgresql_db_username }}"
  DB_PASSWORD: "{{ vault_postgresql_db_password }}"
  DB_HOSTNAME: "{{ vault_postgresql_db_hostname }}"
  SECRET_KEY_BASE: "{{ vault_environment_secret_key_base }}"
  DOMAIN_NAME: "{{ ssl_domain_name }}"
  PORT: 4000
  PHOENIX_OTP_APP_NAME: "{{ phoenix_otp_app_name }}"
  ERLANG_COOKIE: "{{ vault_environment_erlang_cookie }}" 
```
The environment_vars variable is a list of all the environment variables that need to be set for the app you're deploying.  
```
# vault.yml

vault_environment_secret_key_base: "my_secret_key" # Replace with a secret key generated by the mix task
vault_environment_erlang_cookie: "my_erlang_cookie" # Replace with a secure random string 
```
NOTE: The vault_environment_secret_key_base variable can be generated within a Phoenix project using the following command:
```
mix phx.gen.secret
``` 
The vault_environment_erlang_cookie variable can be generated via the erlang_cookie mix task in the [phoenix-starter](https://github.com/CMcDonald82/phoenix-starter) project or can just be set to some secure random string.


## Roles

This section provides a brief explanation of the roles necessary for setting up the server for deployment of a Phoenix app. You do not have to use the exact roles that this playbook uses, you can swap out any role with a similar role based on your preferences (for instance you could swap in a different Nginx role that provides similar functionality).

Variables for each role can be found in /environments/production/all/<role_name>/[vars.yml | vault.yml]

Further explanation for each role can be found by clicking on the role name

* [Users](https://github.com/CMcDonald82/ansible-role-users)
  This role creates/removes users on the Ubuntu system on the server itself. At least one user should be created with this role so that the root user can be prevented from ssh'ing into the server (for obvious security reasons).

* [Postgresql](https://github.com/CMcDonald82/ansible-role-postgresql)
  This role installs and configures PostgreSQL itself (default version 10) as well as any Postgres databases and users specified via the postgresql_databases and postgresql_users variables, respectively. You will need to create at least one database and database user in order to deploy a Phoenix project that uses Ecto to communicate with Postgres.

* [SSL](https://github.com/CMcDonald82/ansible-role-ssl)
  This role installs and uses Certbot to setup SSL encryption certificates free of charge. It's important to run this role BEFORE installing and running Nginx since the Nginx serverblock config used in this playbook is setup for SSL by default and will look for the certs generated by this role.

* [Nginx](https://github.com/CMcDonald82/ansible-role-nginx)
  This role will install and configure Nginx on the server. Nginx will serve static assets and reverse proxy to the Phoenix app. You will need to create a custom serverblock file for the app you will be deploying to the server where Nginx will be installed and running. An example of a custom serverblock file is stored at /templates/nginx/serverblocks/phoenix-example.conf.  

* [Environment](https://github.com/CMcDonald82/ansible-role-environment)
  This role is responsible for setting the environment variables necessary for the Phoenix app to run on the server once it has been deployed. Following are explanations of each of the necessary variables:
  - REPLACE_OS_VARS: This is necessary for environment variables that are written as "${VARIABLE}" to be expanded.
  - DB_NAME: The name of the production database that will be created by the postgresql role. This value MUST match the name of a Postgres database that has been created on this server.
  - DB_USERNAME: This is the username of the user that will be connecting to the DB_NAME database. This user should be created along with the DB_NAME database and be able to connect to the DB_NAME database.
  - DB_PASSWORD: The password for the user that will be connecting to the DB_NAME database. 
  - DB_HOSTNAME: The hostname of the server where the database will be. We can set this to localhost if we are running the database on the same server as the app.
  - SECRET_KEY_BASE: This is a token that is used by the Phoenix app. Generate it with the following command (run this command from within a Phoenix project, then copy the generated key to the SECRET_KEY_BASE variable in the environment_vars variable):
```
mix phx.gen.secret
```
  - DOMAIN_NAME: The domain name for the server you'll be deploying your app to (ex. example.com). You will need to have obtained, setup and configured this separately (purchase domain name and set it up to point to this server you will be deploying your app to). See step 4 of the 'Setup' section of this README, for some information on how to set up a domain name that points to the server you will be deploying to.
  - PORT: The port that the Phoenix app will be running on. You can set this to 4000.
  - PHOENIX_OTP_APP_NAME: This is the name you're giving your project. This variable will be used by the Edeliver config and the custom vm.args.prod file in the Phoenix app. It should be the [snake_case](https://en.wikipedia.org/wiki/Snake_case) version of the name you want to give your new app (for example, if you're deploying a Phoenix app called ExampleApp, you would set this env var to example_app).
  - ERLANG_COOKIE: This cookie is necessary for distributed Erlang apps to communicate with each other (see section 13.7 Security in the [Erlang docs](http://erlang.org/doc/reference_manual/distributed.html). This can be any value, but it is preferred to generate a value via something like Erlang's :crypto module (using the hash/2 and strong_rand_bytes/1 functions, for example). The [phoenix-starter](https://github.com/CMcDonald82/phoenix-starter) project has a mix task called erlang_cookie that does this (feel free to use the phoenix-starter project or just the erlang_cookie task either directly or as inspiration for your own version). 

* [Firewall](https://github.com/CMcDonald82/ansible-role-firewall)
  This role installs and configures UFW to setup a basic firewall on the Ubuntu server

* [Server Security](https://github.com/CMcDonald82/ansible-role-server-security)
  This role sets up some basic server security such as locking out the root user from ssh'ing into the server and disabling password auth. MAKE SURE A USER BESIDES ROOT HAS BEEN CREATED BEFORE RUNNING THIS ROLE, otherwise you can potentially lock yourself out of the server completely. 
