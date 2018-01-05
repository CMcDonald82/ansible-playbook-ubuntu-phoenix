# ansible-playbook-ubuntu-phoenix
Ansible playbook for setting up an Ubuntu server to run a Phoenix (Elixir) app


## Requirements

### Local Machine
* Install [Ansible](http://docs.ansible.com/ansible/latest/intro_installation.html)

* Install the roles listed in install_roles.yml
```
ansible-galaxy install -r install_roles.yml
```

### Target Machine
* Server running Ubuntu 16.04 LTS (Xenial Xerus)


## Setup

* Specify the ip address of the server you wish to deploy to in the hosts file (environments/production/hosts)

NOTE: Feel free to create more environments in the /environments dir - you can name them whatever you want and create separate group_vars & host_vars directories for each one. When adding more environments, be sure to create a hosts file for each one so you can specify the IP address of the server it will be deploying to.

## Using Ansible Vault
NOTE: you can change the editor that is used to open files when running Ansible Vault commands 
See https://www.digitalocean.com/community/tutorials/how-to-use-vault-to-protect-sensitive-ansible-data-on-ubuntu-16-04
ex.) To change the default editor to Sublime Text:
```
export EDITOR='subl -w'
```

* Create a file called .vault_password.txt within the top-level dir. This file will contain a single line which is your private vault password for encrypting/decrypting secrets. I recommend generating a strong one with the following command using the passlib library:
```
python -c "from passlib.hash import sha512_crypt; import getpass; print sha512_crypt.using(rounds=5000).hash(getpass.getpass())"
```
The .vault_password.txt file is ignored in the .gitignore file so it will not be committed to the repo. This helps prevent accidentally distributing sensitive information.

* Create a vault.yml file in environments/production/group_vars/all which will contain the vault_deploy_password variable. Generate the vault_deploy_password using the same script from above (just make sure to run it again to create a new password separate from the one used in .vault_password.txt) Then add it to the vault.yml file:
```
vault_deploy_password: <the password you just generated>
```

* Save the file, then encrypt it:
```
ansible-vault encrypt environments/production/group_vars/all/vault.yml
```

## Usage
* Pass variables to roles. Generally, there are 3 ways that seem to work best:
  1. Place the variables in files in group_vars/all directory (place non-sensitive variables in vars.yml, place sensitive variables such as passwords and secrets in vault.yml (encrypted)). I usually create a separate subdirectory for each role under the group_vars/all directory to keep the variables for each role separate. Each of these role-specific subdirectories contains a vars.yml and vault.yml file.

  2. Use the vars or vars_files entries in the playbook:
  ```
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
  ```

  3. Specify the variables directly with the role:
  ```
  ex.)  
    roles:
      - role: role_name
        variable1: value1
        variable2: value2
        etc...
  ```

* Run the playbook
```
ansible-playbook site.yml
```


## Dependencies
Since these are Python packages, I recommend creating a virtualenv to install these packages in to keep them isolated from the rest of your local environment. See https://virtualenv.pypa.io/en/stable/ for details about virtualenv.

* ansible
* passlib (for generating strong passwords via the command line)