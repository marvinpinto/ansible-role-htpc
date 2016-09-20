htpc
====

[![Build Status](https://img.shields.io/travis/marvinpinto/ansible-role-htpc/master.svg?style=flat-square)](https://travis-ci.org/marvinpinto/ansible-role-htpc)
[![Ansible Galaxy](https://img.shields.io/badge/ansible--galaxy-htpc-blue.svg?style=flat-square)](https://galaxy.ansible.com/marvinpinto/htpc)
[![License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.txt)

This is an Ansible Galaxy meta-role of sorts to install and manage a sample
Home Theatre PC (HTPC) setup.

The goal of this project is to demonstrate how one could use Ansible to wire
together the components needed for an HTPC setup. Wiring all the pieces
together has traditionally been the hardest part so hopefully this will make
things easier.


Features
--------

- [SABnzbd](https://sabnzbd.org/) binary newsgroup downloader
- [SickRage](https://sickrage.github.io/) video library manager for TV shows
- [CouchPotato](https://couchpota.to/) PVR and video library manager for Movies
- [Plex](https://www.plex.tv/) media server for movies, TV shows, music, etc
- [nginx](https://nginx.org/) frontend proxy with Google authentication (via
[OAuth2 Proxy](https://github.com/bitly/oauth2_proxy))
- All designed to be wired together on an Ubuntu 14.04 server


Pre-Requirements
----------------

1. You will need a domain to host this under - this setup uses
   `htpc-sample.example.org` as the example. Update the `htpc_dns_hostname`
   variable with your domain.

1. The default setup here uses Google authentication so you will need to
   register an OAuth Web Application (instructions
   [here](https://github.com/bitly/oauth2_proxy#google-auth-provider)) and
   note the _Client ID_ and _Client Secret_. (corresponding variables
   `htpc_oauth_client_id` and `htpc_oauth_client_secret`)

   You are not restricted to using the Google authentication provider, there
   are [other valid options](https://github.com/bitly/oauth2_proxy) too! Do
   remember to adjust the `oauth2_proxy_cli_args` variable should you choose
   this route.

1. Generate a random 32-byte string using:

   ```bash
   $ date +%s | sha256sum | base64 | head -c 32 ; echo
   ```
   Use this value as your `htpc_oauth_cookie_secret`.

1. Obtain an https [TLS certificate](https://letsencrypt.org/) for the domain
   you chose (`htpc_dns_hostname`). Update the `htpc_tls_cert` and
   `htpc_tls_cert_key` variables with your certficiate and private key
   (respectively).


Role Variables
--------------

The `htpc` specific role variables are all specified in
[defaults/main.yml](defaults/main.yml) while the role-specific meta variables
are in [meta/main.yml](meta/main.yml).


Example
-------

Install this module from Ansible Galaxy into the './roles' directory:
```bash
ansible-galaxy install marvinpinto.htpc -p ./roles
```

Use it in a playbook as follows:
```yaml
- hosts: '127.0.0.1'
  become: true
  roles:
    - role: 'marvinpinto.htpc'
      htpc_dns_hostname: 'htpc-sample.example.org'
      htpc_oauth_client_id: 'your-client-id'
      htpc_oauth_client_secret: 'your-client-secret'
      htpc_oauth_cookie_secret: 'N2U2NTI0NzljNjc2Y2VmNGVlZDZmMDg5'
      htpc_authorized_users_emails: |
        mary.jones@gmail.com
        nancy.drew@gmail.com
      htpc_tls_cert: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      htpc_tls_cert_key: |
        -----BEGIN RSA PRIVATE KEY-----
        ...
        -----END RSA PRIVATE KEY-----
```


Post-setup Configuration
------------------------

### SABnzbd

1. Navigate over to `/sabnzbd` in your browser and follow the initial wizard
   setup.

1. In `/sabnzbd/config/general/`, you probably want to **Disable API-key** (not
   needed since it's running behind OAuth2 Proxy).

1. In `/sabnzbd/config/folders/`:

  - **Temporary Download Folder**: `/opt/downloads/sabnzbd-incomplete`
  - **Completed Download Folder**: `/opt/downloads/misc`
  - **Permissions for completed downloads**: `777`
  - **Scripts folder**: `/opt/nzbtomedia`

1. In `/sabnzbd/config/categories/`:

  - Movie Processing:
    - **Category**: `movies`
    - **Script**: `nzbToCouchPotato.py`
    - **Folder/Path**: `/opt/downloads/unprocessed/movies`

  - TV Processing:
    - **Category**: `tv`
    - **Script**: `nzbToSickBeard.py`
    - **Folder/Path**: `/opt/downloads/unprocessed/tv`


1. In `/sabnzbd/config/switches/`:

  - **Action when encrypted RAR is downloaded**: `abort`
  - **Action when unwanted extension detected**: `abort`
  - **Unwanted extensions**: `exe, com`
  - **Post-Process Only Verified Jobs**: `off`
  - **Ignore Samples**: `on`
  - **Cleanup List**: `nfo, sfv`

1. In `/sabnzbd/config/special/`:

  - `empty_postproc`: `on`

### SickRage

1. Stop the SickRage service:

  ``` bash
  $ service sickrage stop
  ```

1. Allow nginx to proxy requests to SickRage

  ``` bash
  $ sed -i 's/^web_root = ""/web_root = \/sickrage/g' /opt/config/sickrage-config/config.ini
  $ sed -i 's/^handle_reverse_proxy = 0/handle_reverse_proxy = 1/g' /opt/config/sickrage-config/config.ini
  ```

1. Misc. other SickRage settings

  ``` bash
  $ sed -i 's/^use_failed_downloads = 0/use_failed_downloads = 1/g' /opt/config/sickrage-config/config.ini
  $ sed -i 's/^log_nr = 5/log_nr = 1/g' /opt/config/sickrage-config/config.ini
  $ sed -i 's/^auto_update =.*/auto_update = 0/g' /opt/config/sickrage-config/config.ini
  $ sed -i 's/^version_notify =.*/version_notify = 0/g' /opt/config/sickrage-config/config.ini
  $ sed -i 's/^naming_pattern =.*/naming_pattern = %S.N.S%0SE%0E.%E.N/g' /opt/config/sickrage-config/config.ini
  ```

1. Start the SickRage service:

  ``` bash
  $ service sickrage start
  ```


Development
-----------
Use the supplied `Vagrantfile` for local development and testing.

``` bash
$ vagrant up --provision
```
