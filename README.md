# ansible-playbook-ubuntu-phoenix
Ansible playbook for setting up an Ubuntu server to run a Phoenix (Elixir) app


## Requirements

Install the roles listed in install_roles.yml
```
ansible-galaxy install -r install_roles.yml
```

Specify the ip address of the server you wish to deploy to in the hosts file (environments/production/hosts)

NOTE: Feel free to create more environments in the /environments dir - you can name them whatever you want and create separate group_vars & host_vars directories for each one. When adding more environments, be sure to create a hosts file for each one so you can specify the IP address of the server it will be deploying to.

Run the playbook
```
ansible-playbook site.yml
```