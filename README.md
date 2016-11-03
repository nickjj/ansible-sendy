## What is ansible-sendy? [![Build Status](https://secure.travis-ci.org/nickjj/ansible-sendy.png)](http://travis-ci.org/nickjj/ansible-sendy)

It is an [Ansible](http://www.ansible.com/home) role to copy and configure
the self hosted e-mail campaign management software
[Sendy](http://sendy.co/?ref=6L8Qf).

##### Supported platforms:

- Ubuntu 16.04 LTS (Xenial)
- Debian 8 (Jessie)

### What problem does it solve and why is it useful?

[Sendy](http://sendy.co/?ref=6L8Qf) is an excellent choice to manage your e-mail
list without having to pay obnoxious monthly fees.

This role will take care of:

- Create a mysql database and user (use this [MariaDB role](https://github.com/nickjj/ansible-mariadb) for DB installation)
- Copy your local instance of Sendy to a location of your choosing
- Set up permissions as expected for the `uploads/` directory
- Patch `geoip.inc` so you can track e-mails with nginx without getting error 500s
- Set up crontabs for both the `scheduler` and `autoresponder`

## Role variables

Below is a list of default values along with a description of what they do.

```
---

# Where should Sendy be installed to? Example:
#   sendy_install_url: 'https://list.example.com'
sendy_install_url: ''

# Should Sendy create a mysql database and user?
sendy_db_setup: True

# mysql configuration settings.
sendy_db_name: 'sendy'
sendy_db_host: 'localhost'
sendy_db_port: 3306
sendy_db_user: 'sendy'
sendy_db_pass: 'pleasepickabetterpassword'
sendy_db_permissions: '*.*:ALL'
sendy_charset: 'utf8'
sendy_cookie_domain: ''

# Where is Sendy located on the Ansible controller?
sendy_source_path: 'path/to/where/you/extracted/sendy/locally'

# Where do you want Sendy installed to?
sendy_destination_path: '/srv/sendy'

# Should geoip.inc be patched to work for nginx?
sendy_copy_patched_geoip_for_nginx: True

# Which service should restart after configuring Sendy?
sendy_restart_service_name: 'php7.0-fpm'

# These are the recommended cron values by Sendy, the first one is every
# 5 minutes while the second one is every 1 minute.
#
# Do not change these, as per Sendy's instructions.
sendy_cron_scheduled: ['*/5', '*', '*', '*', '*']
sendy_cron_autoresponders: ['*/1', '*', '*', '*', '*']

# Even if your copy of Sendy didn't change, should we force a sync?
sendy_force_sync: False
```

## Example playbook

Let's say you want to set up [Sendy](http://sendy.co/?ref=6L8Qf) to work. Since
this role only handles [Sendy](http://sendy.co/?ref=6L8Qf) specific settings,
it's up to you to install mysql (or [MariaDB](https://github.com/nickjj/ansible-mariadb))
, [PHP](https://github.com/nickjj/ansible-phpfpm) and either
[nginx](https://github.com/nickjj/ansible-nginx) or Apache.

Lucky for you I've already done that, so here's a fully working example.

For the sake of this example let's assume you have a group called **app** and
you have a typical `site.yml` file.

To use this role edit your `site.yml` file to look something like this:

```
---

- name: Configure app server(s)
  hosts: app
  become: True

  roles:
    - { role: nickjj.mariadb, tags: mariadb }
    - { role: nickjj.phpfpm, tags: phpfpm }
    - { role: nickjj.sendy, tags: sendy }
    - { role: nickjj.nginx, tags: nginx }
```

Let's configure all of the above roles. You can do this by opening or creating
`group_vars/app.yml` which is located relative to your `inventory` directory and
then making it look like this:

```
---

mariadb_mysqld_root_password: 'pleasepickastrongerpassword'

sendy_install_url: 'https://list.example.com'
sendy_db_pass: 'pleasepickastrongerpassword'
sendy_source_path: '~/path/to/your/copy/of/sendy'

nginx_sites:
  sendy:
    domains: ['list.example.com']
    root: '/srv/sendy'
    directives:
      - 'index index.html index.php'
    custom_locations: |
      location /l/ {
        rewrite ^/l/([a-zA-Z0-9/]+)$ /l.php?i=$1 last;
      }
      location /t/ {
        rewrite ^/t/([a-zA-Z0-9/]+)$ /t.php?i=$1 last;
      }
      location /w/ {
        rewrite ^/w/([a-zA-Z0-9/]+)$ /w.php?i=$1 last;
      }
      location /unsubscribe/ {
        rewrite ^/unsubscribe/(.*)$ /unsubscribe.php?i=$1 last;
      }
      location /subscribe/ {
        rewrite ^/subscribe/(.*)$ /subscribe.php?i=$1 last;
      }

      location ~ \.php$ {
        try_files $uri $uri/ $uri.php =404;

        include fastcgi_params;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param HTTPS on;
        fastcgi_param modHeadersAvailable true;
        fastcgi_param front_controller_active true;
        fastcgi_pass unix:/var/run/php7.0-fpm.sock;
        fastcgi_intercept_errors on;
        fastcgi_read_timeout 3600;
      }
    custom_root_location_try_files: '$uri $uri/ $uri.php$is_args$args'
```

## Installation

`$ ansible-galaxy install nickjj.sendy`

## Ansible Galaxy

You can find it on the official
[Ansible Galaxy](https://galaxy.ansible.com/nickjj/sendy/) if you want to
rate it.

## License

MIT
