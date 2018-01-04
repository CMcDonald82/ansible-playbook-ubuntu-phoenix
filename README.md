# ansible-playbook-ubuntu-phoenix
Ansible playbook for setting up an Ubuntu server to run a Phoenix (Elixir) app


## Requirements

Install the roles listed in install_roles.yml
```
ansible-galaxy install -r install_roles.yml
```

Specify the ip address of the server you wish to deploy to in 

Run the playbook
```
ansible-playbook site.yml
```