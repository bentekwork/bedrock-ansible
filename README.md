# bedrock-ansible

[![Build Status](https://travis-ci.org/roots/bedrock-ansible.svg)](https://travis-ci.org/roots/bedrock-ansible)

[Ansible](http://www.ansible.com/home) [playbook](http://docs.ansible.com/playbooks.html) designed to be used with [Bedrock](http://roots.io/wordpress-stack/) to configure dev & production servers for Bedrock-based WordPress sites.

This playbook will install the common LEMP (Linux/Nginx/MySQL/PHP) stack with PHP 5.6 (or HHVM) and [MariaDB](https://mariadb.org/) as a drop-in MySQL replacement (but better) on Ubuntu 14.04 Trusty LTS.

[Vagrant](http://www.vagrantup.com) is used to easily start up a Virtual Machine ready for development.

# ToC

* [Requirements](#requirements)
  * [Host OS Notes](#host-os-notes)
* [Installation](#installation)
* [Usage](#usage)
  * [Development](#development)
  * [Staging/Production/Remote servers](#stagingproductionremote-servers)
  * [Passwords](#passwords)
  * [Vagrantfile](#vagrantfile)
* [Vagrant Box](#vagrant-box)
* [Options](#options)
  * [WP Sites](#wp-sites)
  * [Mail](#mail)
* [HHVM](#hhvm)
* [Security](#security)
  * [Locking Down Root](#locking-down-root)
  * [Configuration Files](#configuration-files)
  * [fail2ban and ferm](#fail2ban-and-ferm)
* [Todo](#todo)

## Requirements

* Ansible >= 1.6 - [Installation docs](http://docs.ansible.com/intro_installation.html)
* Virtualbox >= 4.3.10 - [Downloads](https://www.virtualbox.org/wiki/Downloads)
* Vagrant >= 1.5.4 - [Downloads](http://www.vagrantup.com/downloads.html)
* Vagrant-bindfs >= 0.3.1 - [Docs](https://github.com/gael-ian/vagrant-bindfs) (Windows users may skip this)
* Ubuntu 14.04 guest OS

### Host OS Notes

From the Ansible docs:

> Currently Ansible can be run from any machine with Python 2.6 installed (Windows isn’t supported for the control machine).
This includes Red Hat, Debian, CentOS, OS X, any of the BSDs, and so on.

If your host machine is running Windows, the workaround is to run Ansible *on the VM* (since it's running Ubuntu) and not locally. This requires some additions to the default `Vagrantfile` and an extra script.

Example `Vagrantfile` for Windows can be found [here](https://gist.github.com/starise/e90d981b5f9e1e39f632/a4b7819b3663c5d34e7c8a2ce4556b50771073e8). There may also be issues with permissions/UAC and symlinks. See this [comment](https://github.com/roots/bedrock-ansible/issues/8#issuecomment-43346116).

## Installation

1. Download/fork/clone this repo to your local machine.
2. Download/fork/clone [Bedrock](https://github.com/roots/bedrock) or have an existing Bedrock-based site ready.

You should now have the following directories at the same level somewhere:

```
- bedrock-ansible/
- example.dev/
```

Note: The full paths to these directories must not contain spaces or else Ansible will fail. See https://github.com/ansible/ansible/issues/8555.

## Usage

### Development

1. Edit `group_vars/development` and add your WordPress site(s). See [Options](#options) below for details.
2. Optionally add any dev hostnames to your local `/etc/hosts` file (or use the [hostsupdater plugin](https://github.com/cogitatio/vagrant-hostsupdater)).
3. Run `vagrant up`.

Vagrant automatically reads `group_vars/development` and creates a synced folder for each of the sites.

You probably don't need to edit your `Vagrantfile` and should avoid it unless you need more advanced configuration.

Tip: Assuming you leave the default `type: 'nfs'` for `config.vm.synced_folder` (in `Vagrantfile`), see the vagrant docs section ["Root Privilege Requirement"](https://docs.vagrantup.com/v2/synced-folders/nfs.html) for how to avoid having to type your password with every `vagrant up`.

Note: `hosts/development` is there for completeness sake only as Vagrant automatically generates and uses its own.

### Staging/Production/Remote servers

1. Edit `group_vars/<environment/group>` and add your WordPress site(s). See [Options](#options) below for details.
2. Edit `hosts/<environment/group>` and add your server IP(s)/hostname(s).
3. Run `ansible-playbook -i hosts/<environment/group> site.yml`.

Staging example:

1. Edit `group_vars/staging`
2. Edit `hosts/staging`
3. Run `ansible-playbook -i hosts/staging site.yml`

### Passwords

There a few places you'll want to set/change passwords:

* `group_vars/<environment>` - `mysql_root_password`
* `group_vars/<environment>` - `wordpress_sites.admin_password`
* `group_vars/<environment>` - `wordpress_sites.env.db_password`

For staging/production environments, it's best to randomly generate longer passwords using something like [random.org](http://www.random.org/passwords/).

You may be concerned about setting plaintext passwords in a Git repository, and you should be. Any type of server configs such as this playbook should always be in a **private** Git repository.

Even then it's still best to try avoid it if possible, so you have few options:

* Use [Ansible Vault](http://docs.ansible.com/playbooks_vault.html)
* Use [Git Encrypt](https://github.com/shadowhand/git-encrypt)

Note: if you're mostly using this for development environments only, you probably don't need to worry about any of this as everything is just run locally.

### `Vagrantfile`

The example `Vagrantfile` in this project can be kept in this folder, or moved anywhere else such as a project/site folder. Generally if you want to have multiple sites on 1 Vagrant VM, you should keep the `Vagrantfile` where it is (in the bedrock-ansible dir). If you want to have 1 Vagrant VM *PER* project/site, you should make copies of the `Vagrantfile` and put them into each project's dir. You'd then run `vagrant up` from the project specific directory.

Whenever you move or copy the `Vagrantfile` somewhere else, you need to make sure to adjust `ANSIBLE_PATH` (in `Vagrantfile`) and `local_path` (in `group_vars/development`).

## Vagrant Box

By default, the example `Vagrantfile` now uses the `roots/bedrock` box. It's publicly available on the Vagrant Cloud site [here](https://vagrantcloud.com/roots/bedrock).

The `roots/bedrock` box is simply the regular `ubuntu/trusty64` base box already provisioned with this playbook (except for the `wordpress-sites` role). The benefit to using this base box instead of a bare Ubuntu one is that provisioning will be much faster.

Vagrant Cloud offers releases/versions for the boxes, so the `roots/bedrock` box versions will be kept in sync with this project. You can see if there's updates by running `vagrant box outdated` and update it with `vagrant box update`.

Note: you can always set the box back to the base Ubuntu one if you prefer with `config.vm.box = 'ubuntu/trusty64'`

## HHVM

[HHVM](http://hhvm.com/) can be used instead of PHP 5.6 by setting `hhvm: true` in `group_vars/all`.

The only caveat is that it's not set up to seamlessly switch between the two on already provisioned machines. You'll either need to do some manual cleanup (like removing php5.6) or just re-create your machine.

See `roles/hhvm/README.md` for more information on running HHVM with WordPress.

## Options

All Ansible configuration is done in [YAML](http://en.wikipedia.org/wiki/YAML).

### WP Sites

`wordpress_sites` is the top level array used to define the WordPress sites/virtual hosts that will be created.

* `site_name` (required) - name used to identify site (commonly the domain name) (default: none)
* `site_hosts` (required) - array of hosts that Nginx will listen on (default: none)
* `user` (optional) - user owner of site directories/files (default: `root` | `user` in `site.yml`)
* `group` (optional) - group owner of site directories/files (default: `www-data`)
* `site_install` (optional) - whether to install WordPress or not (default: `true`)
* `site_title` (optional) - WP site title (default: `site_name`)
* `db_create` (optional) - whether to auto create a database or not (default: `true`)
* `db_import` (optional) - Path to local `sql` dump file which will be imported (default: `false`)
* `system_cron` (optional) - Disable WP cron and use system's (default: `false`)
* `run_composer` (optional) - Run `composer install` before WP install (default: `true`)
* `admin_user` (optional) - WP admin user name (default: `admin`)
* `admin_password` (required if `site_install`) - WP admin user password (default: none)
* `admin_email` (required if `site_install`) - WP admin email address (default: none)
* `multisite` (optional) - hash of multisite options
  * `enabled` (optional) - Multisite enabled flag (default: `false`)
  * `subdomains` (optional) - subdomains option (default: `false`)
  * `base_path` (optional) - base path/current site path (default: `/`)
* `env` (required) - hash of multisite options
  * `wp_home` (required) - `WP_HOME` constant or `home` option (default: none)
  * `wp_siteurl` (required) - `WP_SITEURL` constant or `siteurl` option (default: none)
  * `wp_env` (required) - WordPress environment (default: none)
  * `db_name` (optional) - name of database (default: `site_name`)
  * `db_user` (required) - database user name (default: none)
  * `db_password` (required) - database user password (default: none)
  * `db_host` (required) - database host (default: `localhost`)

### Mail

Outgoing mail is done by the ssmtp role. ssmtp is a lightweight SMTP mail relay basically. In order to send external emails, you'll need to configure an SMTP server.

We always suggest using an external email service rather than your own because it's very difficult to setup a proper email server.

Some suggested services:

* [Mandrill](http://mandrill.com/)
* [Sendgrid](https://sendgrid.com/)
* [Mailgun](http://www.mailgun.com/)
* [Amazon SES](http://aws.amazon.com/ses/)

All of these offer around 10k+ emails for free per month. Once you have SMTP credentials, configure them in `group_vars/all`.

* `mail_smtp_server`: hostname:port
* `mail_hostname`: hostname for mail delivery
* `mail_user`: username
* `mail_password`: password (or API key)

## Security

### Locking down `root`

By default, ssh access to the `root` user is allowed on fresh server installs. The `secure-root.yml` playbook does a few things to lock the server down:

* Creates users with `sudo` privileges.
* Disable ssh root login.
* Disable password authentication for ssh access.
* Set better ssh defaults.

There are a few variables to be aware of:

* `sudoers` located in `group_vars/all`

This variable contains a list of dictionaries of users to create.
* `sudoer_passwords` located in `vars/sudoer_passwords`. This is a dictionary of user => password pairs used when creating users with sudo privileges.

Here's an example list of `sudoers`:

```yaml
sudoers:
  - user: "admin"
    groups: ["sudo"]
  - user: "another_user"
    groups: ["sudo"]
```

`user` is the key to be used when creating the user and `groups` is an array used to determine the groups the user belongs to. The first item in the array will be used when setting the user's primary group.

Here's an example list of `sudoer_passwords`:

```yaml
sudoers:
  admin: $6$rounds=100000$JUkj1d3hCa6uFp6R$3rZ8jImyCpTP40e4I5APx7SbBvDCM8fB6GP/IGOrsk/GEUTUhl1i/Q2JNOpj9ashLpkgaCxqMqbFKdZdmAh26/
  another_user: $6$rounds=100000$r3ZZsk/uc31cAxQT$YHMkmKrwgXr3u1YgrSvg0wHZg5IM6MLEzqOraIXqh5o7aWshxD.QaNeCcUX3KInqzTqaqN3qzo9nvc/QI0M1C.
```

The passwords were generated using the python command [found here](http://docs.ansible.com/faq.html#how-do-i-generate-crypted-passwords-for-the-user-module). The passwords generated here are `example_password` and `another_password`, respectively. The ansible user module doesn't handle any encryption and passwords must be encrypted beforehand. It's also recommended `vars/sudoer_passwords.yml` be encrypted using one of the encryption methods described in the [passwords](#passwords) section of this document. Passwords are stored separately in order to ease the separation of encrypted var files and are looked up based on the user name.

This playbook should be run on remote hosts and is designed to be run once whenever the server is initially created (as the root user will be inaccessible after running and the ansible default user is the [current user](http://docs.ansible.com/intro_configuration.html#remote-user)). To invoke, run `ansible-playbook -i hosts/ENVIRONMENT secure-root.yml`. Because the root user is locked down, remote hosts also need to use a different ssh user. You can set the default remote user in `site.yml` by setting the `remote_user` variable. All subsequent provisions need to have the `--ask-sudo-pass` flag set when using `ansible-playbook`.

### `fail2ban` and `ferm`

> Fail2ban scans log files (e.g. /var/log/apache/error_log) and bans IPs that show the malicious signs -- too many password failures, seeking for exploits, etc.

From http://www.fail2ban.org/wiki/index.php/Main_Page

> ferm is a tool to maintain complex firewalls, without having the trouble to rewrite the complex rules over and over again. ferm allows the entire firewall rule set to be stored in a separate file, and to be loaded with one command.

From http://ferm.foo-projects.org/

You can read the role documentation for `fail2ban` [here](roles/fail2ban/README.md) and `ferm` [here](roles/ferm/README.md).

## Todo

* Multisite: basic support is included but not yet complete. There are issues with doing a network install from scratch via WP-CLI.
* Nginx: configuration needs more options and advanced setups like static files and subdomain multisite support.
