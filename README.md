
  
Automate the process of creating and provisioning cloud virtual machines with a complete, running instance of WordPress.

## Prerequisites

To make use of these files, you'll need to have the following prerequisites installed on your workstation:

* [Ansible](https://www.virtualbox.org/)

Note that these scripts are designed for use with Ubuntu, so they make use of the apt packaging manager and other conventions Debian-derived distributions share (filesystem locations, configuration file structure, etc.). **This means they will only work properly out of the box with Debian Linux or Debian-derived distributions such as Ubuntu.** If you use Red Hat/CentOS/Fedora, or another distribution that uses a different package manager, consult the Ansible documentation for instructions on how to modify these scripts to replace the Debian-based commands commands to those for your particular package manager and distro.


Cloud service providers tested:

* [Vultr](https://vultr.com/)

Be certain that you have both public and private ssh rsa keys present in your home directory (~/.ssh/id_rsa, ~/.ssh/id_rsa.pub)

After obtaining a Vultr API key, and allowing subnet access (see https://my.vultr.com/settings/#settingsapi), test with Ansible to verify account connection:
```sh
VULTR_API_KEY=7TLL77DG3HU6BCQ3BOHPF65HOICUQATCV6VQ ansible -m vultr_account_facts localhost
```
The Ansible vault encrypted var 'vultr_api_key' contains an API key like the one shown above. You'll need to know the vault password stored on LastPass to deploy.

Do a trial run of the server create process for the vultr-dev-1 instance:
```sh
ansible-playbook playbooks/server.yml --inventory hosts --limit vultr-dev-1 --tags create_server --check --diff -v --ask-vault-pass
```

Create a server (it takes about 5 minutes for a server to be created):
```sh
ansible-playbook playbooks/server.yml --inventory hosts --limit vultr-dev-1 --tags create_server -v --ask-vault-pass 
```

See if the server is running:
```sh
ansible-playbook playbooks/server.yml --inventory hosts --limit vultr-dev-1 -v --ask-vault-pass
```

Destroy the server instance when it is no longer needed. Note that **all data is lost** including applications and database content. The assigned IP address is also relinquished.
```sh
ansible-playbook playbooks/server.yml --inventory hosts --limit vultr-dev-1 --tags destroy_server -v --ask-vault-pass
```

It's possible to encrypt a variable, in this case the vultr_api_key and paste the result into a yml file:
```sh
ansible-vault encrypt_string '7TLL77DG3HU6BCQ3BOHPF65HOICUQATCV6VQ' --name 'vultr_api_key'
New Vault password: 
Confirm New Vault password: 
vultr_api_key: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66643630663136363135383063353865643465623961333839653338663639333231323232366162
          3838383530306630626533623462343637393362303866360a636234633238356138333561373638
          32343737303566623139333430386532303230656132386663353436633466306436353032626232
          3763313134393338640a656230656464336431383063633835323032323934386235356266346466
          64363263323032363338643930316564303133663030376165656132646332373036626538623134
          3439323263373739623564373362613436353262393139353261
Encryption successful
```

## Install Apache, MySQL, and PHP

Once you've got an Ubuntu server functioning in the cloud install Apache, MySQL, and PHP
```sh
ansible-playbook playbooks/lamp.yml --inventory hosts --limit vultr-dev-1 -v --ask-vault-pass
```
Make note of the generated MySQL password.

## Install Wordpress

```sh
ansible-playbook playbooks/wordpress.yml --inventory hosts --limit vultr-dev-1 -v --ask-vault-pass

```
Save the generated Wordpress admin password for wp-login access.
