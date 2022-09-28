
  
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

## Install a Wordpress site

```sh
ansible-playbook playbooks/wordpress.yml --inventory hosts --limit vultr-dev-1 -v --extra-vars wordpress_site=sundogdev --ask-vault-pass
```

In the above example the Wordpress playbook installs a specified Wordpress site (sundogdev) on the host vultr-dev-1. The site parameters must be defined in the roles/wordpress/vars/main.yml file for the playbook to run successfully.
```sh
  sundogdev:
    domain: dev.sundog-studios.com
    db_prefix: sds
    admin_email: info@example.com
    title: Sundog Studios
    plugins:
      - advanced-custom-fields
      - advanced-gutenberg-blocks
      - atomic-blocks
      - editor-blocks
      - kadence-blocks
      - regenerate-thumbnails
      - ultimate-addons-for-gutenberg
      - wpforms-lite
      - wordfence
      - wordpress-seo
    themes:
      - astra
```

The Wordpress playbook
* installs wp-cli
* creates the mysql database for the site
* installs Wordpress and its specified themes and plugins
* creates the http config files
* if a domain is specified gets SSL certificate from certificate authority using Let's Encrypt

When the playbook completes be sure to save the generated Wordpress admin password for wp-login access.

## Backup a Wordpress site from server to local workstation's /var/tmp/wpfish/backup directory

```sh
ansible-playbook playbooks/wordpress-backup.yml --inventory hosts --limit fish -v --extra-vars wordpress_site=sundog
```

## Restore a Wordpress site from local backup to server 

```sh
ansible-playbook playbooks/wordpress-restore.yml --inventory hosts --limit fish -v --extra-vars "wordpress_site=sundogdev from_wordpress_site=sundog"
```

The sundog site must exist and have been backed up to /var/tmp/wpfish/backup/sundog on the local workstation before a restore can be performed. The sundogdev site must also exist prior to a restore.

## Rollback a Wordpress site immediately after a restore was performed

```sh
ansible-playbook playbooks/wordpress-rollback.yml --inventory hosts --limit fish -v --extra-vars wordpress_site=sundogdev
```

If the sundogdev site restore from the sundog backup didn't give the desired results, rollback to the way sundogdev was before the restore using the /var/tmp/wpfish/backup/sundogdev directory and /var/tmp/wpfish/backup/sundogdev.sql database

## Import an older Wordpress site from another server that was originally not in this Ansible playbook 
* Obtain backups of both the Wordpress files and database export (.sql file) for the site.
* Add the site to roles/wordpress/vars/main.yml 
* Rename the directory containing the Wordpress files and the database.sql file to match the site name added to roles/wordpress/vars/main.yml
* Put the renamed directory and sql file into the local import directory
* Before creating the new Wordpress site, at your domain hosting service change the DNS & Nameservers to point the new server (e.g. ns1.vultr.com and ns2.vultr.com)
* In your Vultr account add a new DNS entry for the new site (e.g. dev.sundog.com)
* Create the new Wordpress site
* Import the old Wordpress site to the new Wordpress site that you just created

```sh
  sundogOLD:
    domain: sundog.com
    db_prefix: sds 
    admin_email: admin@example.com
    title: "Sundog Studios"
```

```sh
mv sundog /var/tmp/import/sundogOLD
mv sundog_production.sql /var/tmp/import/sundogOLD.sql
```

```sh
ansible-playbook playbooks/wordpress.yml --inventory hosts --limit fish -v --extra-vars wordpress_site=sundogdev --ask-vault-pass
```

```sh
ansible-playbook playbooks/wordpress-import.yml --inventory hosts --limit fish -v --extra-vars "wordpress_site=sundogdev from_wordpress_site=sundogOLD"
```
