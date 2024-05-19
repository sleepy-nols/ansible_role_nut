# Ansible Network UPS Tools

Ansible role to install and configure [NUT](https://networkupstools.org) on Debian-like systems, plus automatically connecting servers and clients together.

This role configures part of its automation over the Ansible inventory. You need to therefore configure it correct for this role to work. The [example configuration](/#) is a minimum set that needs to exist.

This role uses the terminology `server` and `client` instead of `master` and `slave`.


---

What it does:
- automatic connection of clientsa and servers
- automatic instance mode selection
- currently only supports 1 ups per server

Assumptions made:
- clients can reach server via its default ip address (e.g. local network or vpn)


Client and server devices are seperated into two inventory groups: `[nut_clients]` and `[nut_servers]`. Accordingly client hosts should be present in the inventory group `[nut_clients]` and server hosts in `[nut_server]`.


Host enters configuration mode when:

**standalone:**
- the host is in inventory group `[nut_servers]`
- no host that is member of inventory group `[nut_clients]` marks the host as `nut_server_member=<the_host>`

**netserver:**
- the host is in inventory group `[nut_servers]`
- one ore more hosts that are member of inventory group `[nut_clients]` mark the host as `nut_server_member=<the_host>`

**netclient:**
- the host is in inventory group `[nut_clients]`
- the host marks a host which is member of inventory group `[nut_servers]` with `nut_server_member=<a_host>`


---

## Example configuration

```yml
# inventory/host_vars/nut-server01.yml
nut_ups_devices:
  mydevice: # <-- unique device identifyer
    driver: "usbhid-ups" # https://networkupstools.org/stable-hcl.html
    port: "auto"
    desc: "some description"
```
Example **server** hostvars file.
<br>
<br>

```yml
# inventory/nut

[nut_all:children]
nut_servers
nut_clients

[nut_servers]
nut-server01 nut_master_password="secretpassword"

[nut_clients]
nut-client01 nut_server_member="nut-server01" nut_password="secretpassword"
nut-client02 nut_server_member="nut-server01" nut_password="secretpassword"

```
Example **inventory** file. Variables can alternatively be set in hostvars.
<br>
<br>


```yml
# playbooks/nut.yml
---
- name: Build variables locally
  hosts: localhost
  pre_tasks:

    - name: Prepare variables
      ansible.builtin.set_fact:
        nut_server_client_map: {}
        nut_client_list: []

    - name: block
      block:
        - name: Build checks
          ansible.builtin.set_fact:
            test: "{% for client in groups['nut_clients'] %}{% if hostvars[client]['nut_server_member'] %}{% endif %}{% endfor %}"
      rescue:
        - assert:
            that: false
            msg: "A nut client is probably missing the nut_server_member variable."
#
    - name: Build server - client tree
      ansible.builtin.set_fact:
        nut_server_client_map: "{% for client in groups['nut_clients'] %}{% if hostvars[client]['nut_server_member'] == item %}{% set _ = nut_client_list.append(client) %}{% endif %}{% endfor %}{{ {item: nut_client_list} | combine(nut_server_client_map) }}"
      loop: "{{ groups['nut_servers']  }}"

    - name: Show server - client tree
      debug:
        msg: "{{ nut_server_client_map }}"

- name: "Install and configure NUT on servers and clients"
  hosts: nut_all
  roles:
    - ansible_role_nut
```
Example **playbook**.
<br>
<br>

---

## License

GPLv3
