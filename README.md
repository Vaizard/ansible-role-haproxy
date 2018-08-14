# Ansible Role: HAProxy

[![Build Status](https://travis-ci.org/vaizard/mage-haproxy.svg?branch=master)](https://travis-ci.org/vaizard/mage-haproxy)

Installs HAProxy on RedHat/CentOS and Debian/Ubuntu Linux servers. This fork has been tested well on Ubuntu 18.04 with HAProxy 1.8.

## Optional roles

- https://github.com/vaizard/mage-base-common if you'd like to use a common setup base, which also includes dhparm and snakeoil certs generation. Include with `role: mage-base-common` before this role in your playbook.
- https://github.com/openmicroscopy/ansible-role-selinux-utils if SELinux is enabled on CentOS 7 and you are using non-standard ports. Include with `role: openmicroscopy.selinux-utils` before this role in your playbook.
- https://github.com/vaizard/mage-certbot if you want to use Letsencrypt certificates. Include with `role: mage-certbot` after this role in your playbook.


## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

    haproxy_socket: /var/lib/haproxy/stats

The socket through which HAProxy can communicate (for admin purposes or statistics). To disable/remove this directive, set `haproxy_socket: ''` (an empty string).

    haproxy_chroot: /var/lib/haproxy

The jail directory where chroot() will be performed before dropping privileges. To disable/remove this directive, set `haproxy_chroot: ''` (an empty string). Only change this if you know what you're doing!

    haproxy_user: haproxy
    haproxy_group: haproxy

The user and group under which HAProxy should run. Only change this if you know what you're doing!

    haproxy_frontends:
    - name: 'hafrontend'
      address: '*:80'
      #bind_params: 'param1 param2'
      #extra bind parameters, 'ssl' for example
      mode: 'http'
      #params:
      #  - 'some extra frontend param, acl for example'
      backend: 'habackend'
      # Optional:
      timeout client: 10s
    - name: 'hassl'
      address: 
        - '*:443'
        - '*:8443'
      bind_params: 'ssl'
      mode: 'http'
      backend: 'sslhabackend'

List of HAProxy frontends.

    haproxy_backends:
    - name: 'habackend'
      # All fields are optional apart from servers
      mode: 'http'
      balance_method: 'roundrobin'
      options:
        - "haproxy_backend_httpchk: 'HEAD / HTTP/1.1\r\nHost:localhost'"
      params:
        - 'stick on src'
      servers:
      - name: app1
        address: 192.168.0.1:80
	    extra_opts: 'inter 2s'
      - name: app2
        address: 192.168.0.2:80
      timeout connect 5s
      timeout server 20s
    - name: 'sslhabackend'
      servers:
      - name: sslapp
        address: 192.168.0.3:443

List of HAProxy backends and servers to which HAProxy will distribute requests.

    haproxy_global_vars:
      - 'ssl-default-bind-ciphers ABCD+KLMJ:...'
      - 'ssl-default-bind-options no-sslv3'

A list of extra global variables to add to the global configuration section inside `haproxy.cfg`.

Advanced users can override the template used for `haproxy.cfg` by setting `haproxy_cfg_template`. In this case most of the above role variables will be ignored unless the default template is copied.

## Example Playbook

    - hosts: balancer
      sudo: yes
      roles:
        - { role: openmicroscopy.haproxy }

## Using this role with letsencrypt

A reliable haproxy+letsencrypt ansible-managed setup can be achieved by using this role along with https://github.com/vaizard/mage-certbot (another fork of Jeff Geerling's awesome work). Below is an example configuration of frontends 
(with rules for /.well-known/acme-challenge/) as well as required letsencrypt backends (port 8888 for the http-01 challenge). The rest is a (relatively) sane lamp-stack configuration example.

```yaml
### base-common configuration
dhparam_size: 2048

### letsencrypt
certbot_certs:
  - domains: 
      - ct01.example.com


### haproxy frontends
haproxy_frontends:
  - name: fe_http
    address: '*:80'
    backend: 'be_lamp'
    params:
      - 'acl is_certbot path_beg -i /.well-known/acme-challenge/'
      - 'use_backend be_certbot if is_certbot'
      - 'redirect scheme https code 301 if !{ ssl_fc } !is_certbot' 
  - name: fe_https
    address: '*:443'
    bind_params: ssl crt-list /etc/haproxy/crt-list.txt
    params:
      - 'acl is_certbot_ssl path_beg -i /.well-known/acme-challenge/'
      - 'use_backend be_certbot_ssl if is_certbot_ssl'
    backend: 'be_lamp'


### haproxy backends
haproxy_backends:
  - name: be_lamp
    check: false
    options:
      - 'forwardfor'
      - 'http-server-close'
      #- "httpchk HEAD / HTTP/1.1\r\nHost:localhost"  # perform healthchecks
    params:
      - 'http-request set-header X-Forwarded-Port %[dst_port]'
      - 'http-request add-header X-Forwarded-Proto https if { ssl_fc }'
      - 'compression algo gzip'
      - 'compression type text/css text/javascript text/xml text/plain text/x-component application/javascript application/json application/xml application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml'
    servers:
      - name: srv_lamp
        address: "ct01-lamp.lxd:80"
  - name: be_certbot
    check: false
    servers:
      - name: srv_certbot
        address: 0.0.0.0:8888
  - name: be_certbot_ssl
    check: false
    servers:
      - name: srv_certbot_ssl
        address: 0.0.0.0:8889

# https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy
haproxy_global_vars:
  - 'tune.ssl.default-dh-param {{ dhparam_size }}'
  - 'ssl-default-bind-ciphers ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS'
  - 'ssl-default-bind-options no-sslv3 no-tls-tickets'
  - 'ssl-default-server-ciphers ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS'  
  - 'ssl-default-server-options no-sslv3 no-tls-tickets'

```

## License

MIT / BSD

## Author Information

This role was created in 2015 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
Further work comes from the [Openmicroscopy team](https://github.com/openmicroscopy/ansible-role-haproxy) and all contributors.




