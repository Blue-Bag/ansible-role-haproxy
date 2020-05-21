# Ansible Role: HAProxy

[![Build Status](https://travis-ci.org/Blue-Bag/ansible-role-haproxy.svg?branch=master)](https://travis-ci.org/Blue-Bag/ansible-role-haproxy)
[![Ansible Role](https://img.shields.io/ansible/role/41401.svg)](https://galaxy.ansible.com/ome/haproxy/)

Installs HAProxy on RedHat/CentOS and Debian/Ubuntu Linux servers.

**Note**: This role _officially_ supports HAProxy versions 1.5 and 1.6. Future versions may require some rework.

**Note**: This role combines two forks on originating from Jeff Geerling and the other from https://github.com/Oefenweb/ansible-haproxy

## Requirements

If SELinux is enabled on CentOS 7 and you are using non-standard ports you must include `role: ome.selinux_utils` before this role in your playbook.


## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

    haproxy_socket: /var/lib/haproxy/stats

The socket through which HAProxy can communicate (for admin purposes or statistics). To disable/remove this directive, set `haproxy_socket: ''` (an empty string).

    haproxy_chroot: /var/lib/haproxy

The jail directory where chroot() will be performed before dropping privileges. To disable/remove this directive, set `haproxy_global_chroot: ''` (an empty string). Only change this if you know what you're doing!

    haproxy_global_user: haproxy
    haproxy_global_group: haproxy

The user and group under which HAProxy should run. Only change this if you know what you're doing!

    haproxy_frontends:
    - name: 'hafrontend'
      address: '*:80'
      extra_addresses:
        - '*:8080'
      #bind_params: 'param1 param2'
      #extra bind parameters, 'ssl' for example
      mode: 'http'
      #params:
      #  - 'some extra frontend param, acl for example'
      backend: 'habackend'
      # Optional:
      timeout_client: 10s
      setsecurecookies: yes
      set_hsts: yes

List of HAProxy frontends.

    haproxy_backends:
    - name: 'habackend'
      # All fields are optional apart from servers
      mode: 'http'
      balance_method: 'roundrobin'
      options:
        - "haproxy_backend_httpchk: 'HEAD / HTTP/1.1\r\nHost:localhost'"
      params:
        - 'stick-table type ip size 200k expire 30m'
        - 'stick on src'
      servers:
      - name: app1
        address: 192.168.0.1:80
	    extra_opts: 'inter 2s'
      - name: app2
        address: 192.168.0.2:80
      timeout_connect 5s
      timeout_server 20s

List of HAProxy backends and servers to which HAProxy will distribute requests.

    haproxy_global_vars:
      - 'stats           socket /var/run/haproxy.stat mode 777'
      - 'spread-checks   5'

Note that the ssl defaults are now handled specifically with ha_proxy_set_ssl_default


A list of extra global variables to add to the global configuration section inside `haproxy.cfg`.

Advanced users can override the template used for `haproxy.cfg` by setting `haproxy_cfg_template`. In this case most of the above role variables will be ignored unless the default template is copied.

# SSL Map
* `haproxy_ssl_map`: [default: `[]`]: SSL declarations
* `haproxy_ssl_map.{n}.src`: The local path of the file to copy, can be absolute or relative (e.g. `../../../files/haproxy/etc/haproxy/ssl/star-example-com.pem`)
* `haproxy_ssl_map.{n}.dest`: The remote path of the file to copy (e.g. `/etc/haproxy/ssl/star-example-com.pem`)
* `haproxy_ssl_map.{n}.owner`: The name of the user that should own the file (optional, default `root`)
* `haproxy_ssl_map.{n}.group`: The name of the group that should own the file (optional, default `root`)
* `haproxy_ssl_map.{n}.mode`: The mode of the file, such as 0400 (optional, default `0400`)

``` Yaml
haproxy_ssl_map:
      - src: ../../../files/haproxy/etc/haproxy/ssl/star-example0-com.pem
        dest: /etc/ssl/private/star-example0-com.pem
      - src: ../../../files/haproxy/etc/haproxy/ssl/star-example1-com.pem
        dest: /etc/ssl/private/star-example1-com.pem
      - src: ../../../files/haproxy/etc/haproxy/ssl/star-example2-com.pem
        dest: /etc/ssl/private/star-example2-com.pem
```

# Users
* `haproxy_userlists`: [default: `[]`]: Userlist declarations
* `haproxy_userlists.{n}.name`: [required]: The name of the userlist
* `haproxy_userlists.{n}.users`: [required] Userlist users declarations
* `haproxy_userlists.{n}.users.{n}.name`: [required] The username of this user
* `haproxy_userlists.{n}.users.{n}.password`: [optional] Password hash of this user. **One of `password` or `insecure_password` must be set**
* `haproxy_userlists.{n}.users.{n}.insecure_password`: [optional] Plaintext password of this user. **One of `password` or `insecure_password` must be set**
* `haproxy_userlists.{n}.users.{n}.groups`: [optional] List of groups to add the user to

```Yaml
   haproxy_userlists:
      - name: 'Admin Users'
        users:
          - name: admin
            password: $hash
            insecure_password: if hashnotprovided
            groups:
              - admin_group
      - name: 'Guest Users'
        users:
          - name: guest
            password: $hash
            insecure_password: if hashnotprovided
            groups:
              - guest_group
```

* `haproxy_resolvers`: [default: `[]`]: Resolvers (name servers) declarations
* `haproxy_resolvers.{n}.name`: [required]: The name of the name server list
* `haproxy_resolvers.{n}.nameservers`: [required] list of DNS servers
* `haproxy_resolvers.{n}.nameservers.{n}.name`: [required] label of the server, should be unique
* `haproxy_resolvers.{n}.nameservers.{n}.listen`: [required] Defines a listening address and/or ports, e.g. `8.8.8.8:53`
* `haproxy_resolvers.{n}.accepted_payload_size`: [optional]: Defines the maximum payload size (in bytes) accepted by HAProxy and announced to all the name servers configured in this resolvers section. If not set, HAProxy announces 512. (minimal value defined by RFC 6891)
* `haproxy_resolvers.{n}.parse_resolv_conf`: [optional]: If set to `true`, adds all nameservers found in `/etc/resolv.conf` to this resolver's nameservers list.
* `haproxy_resolvers.{n}.resolve_retries`: [optional]: Defines the number of queries to send to resolve a server name before giving up.
* `haproxy_resolvers.{n}.hold`: [optional]: A list of directives defining `<period>` during which the last name resolution should be kept based on last resolution `<status>`.
* `haproxy_resolvers.{n}.hold.{status}`: [optional]: hold directives in `<status>:<period>` format. Key must be one of (`nx`, `other`, `refused`, `timeout`, `valid`, `obsolete`). Value is interval between two successive name resolutions in HAProxy time format.
* `haproxy_resolvers.{n}.timeout`: [optional]: Defines timeouts related to name resolution
* `haproxy_resolvers.{n}.timeout.{event}`: [optional]: timeout directives in `<event>:<time>` format. Key must be one of (`resolve`, `retry`). Value is time related to the event in the HAProxy time format.

* `haproxy_acl_files`: [default: `[]`]: ACL file declarations
* `haproxy_acl_files.{n}.dest`: [required]: The remote path of the file (e.g. `/etc/haproxy/acl/api.map`)
* `haproxy_acl_files.{n}.content`: [default: `[]`]: The content (lines) of the file (e.g. `['v1.0 be_alpha', 'v1.1 be_bravo']`)

# Stat listener
# stats
# https://cbonte.github.io/haproxy-dconv/1.8/configuration.html#4-stats%20admin
```Yaml
haproxy_listen:
  - name: stats
    description: Global statistics
    bind:
      - listen: '*:9090'
        param:
          - ssl
          - 'crt example.com.pem'
    mode: http
    stats:
      enable: true
      uri: /
      options:
        - hide-version
        - show-node
      admin: if TRUE
      refresh: 5s
      realm: 'Strictly\ Private'
      auth:
        - user: admin
          passwd: 'NqXgKWQ9f9Et'
```
## Dependencies

None.

## Example Playbook

    - hosts: balancer
      sudo: yes
      roles:
        - { role: ome.haproxy }

## License

MIT / BSD

## Author Information

This role was created in 2015 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
Forked from the fork:
Pulled in alot of goodies from Oefenweb.ansible-haproxy
