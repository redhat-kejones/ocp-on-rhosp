## Ansible Host Setup

You need to have a host for which to run the playbooks from. Assuming a RHEL 7.5+ box

Configure the proper repos and install the needed packages

```
[root@ansible-host ~]# subscription-manager repos --disable=* --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-7-server-ansible-2.6-rpms --enable=rhel-7-server-ose-3.11-rpms --enable=rhel-7-server-openstack-14-rpms

[root@ansible-host ~]# yum install -y git ansible python-openstackclient python-heatclient openshift-ansible python2-shade python-dns python-virtualenv
```

Clone this repo so we have a baseline to operate from
```
[root@ansible-host ~]# git clone https://github.com/redhat-kejones/ocp-on-rhosp.git
[root@ansible-host ~]# cd ocp-on-rhosp
```

In order to control what libraries get installed and their dependencies, I'll use virtualenv
```
[root@ansible-host ocp-on-rhosp]# virtualenv ./
[root@ansible-host ocp-on-rhosp]# . bin/activate
(ocp-on-rhosp) [root@ansible-host ocp-on-rhosp]# pip install shade
(ocp-on-rhosp) [root@ansible-host ocp-on-rhosp]# pip install dnspython
(ocp-on-rhosp) [root@ansible-host ocp-on-rhosp]# pip install jinja2
```

Copy a few pieces from the openshift-ansible project
```
(ocp-on-rhosp) [root@ansible-host ocp-on-rhosp]# cp -a /usr/share/ansible/openshift-ansible/playbooks/openstack/sample-inventory ./inventory
(ocp-on-rhosp) [root@ansible-host ocp-on-rhosp]# cp /usr/share/ansible/openshift-ansible/playbooks/openstack/ansible.cfg ./
```

Add any_errors_fatal setting to ansible.cfg [defaults] section
```
(ocp-on-rhosp) [root@ansible-host ocp-on-rhosp]# vi ansible.cfg
...
[defaults]
...
any_errors_fatal = true
...
```

Make sure that you have an OpenStack rc file to auth to your OpenStack cloud. I got my operatorrc file from my undercloud. This authenticates me as operator user on my RHOSP 14 overcloud
```
(ocp-on-rhosp) [root@ansible-host ocp-on-rhosp]# scp stack@192.168.0.5:operatorrc ./
(ocp-on-rhosp) [root@ansible-host ocp-on-rhosp]# source operatorrc
```

Now I should be able to run openstack commands and get responses
```
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# openstack server list
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# openstack image list
```

You need to now edit your all group_vars file
```
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# vi inventory/group_vars/all.yml
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# vi inventory/group_vars/OSEv3.yml
```

## Running openshift-ansible OpenStack playbooks

Now you are ready to run the playbooks.

If you just want to run the provision piece for OpenStack Heat stack creation
```
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# ansible-playbook -i /usr/share/ansible/openshift-ansible/playbooks/openstack/inventory.py -i inventory /usr/share/ansible/openshift-ansible/playbooks/openstack/openshift-cluster/provision.yml
```

> NOTE: in between provision and installation, you should set up all forward and reverse DNS records
>       if you are not using the mod later in this README to do IPA/IdM DNS population

After heat stack creation and DNS setup, you can run the install piece for OpenShift installation
```
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# ansible-playbook -i /usr/share/ansible/openshift-ansible/playbooks/openstack/inventory.py -i inventory /usr/share/ansible/openshift-ansible/playbooks/openstack/openshift-cluster/install.yml
```

If you want to run it all in one go, do the following
```
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# ansible-playbook -i /usr/share/ansible/openshift-ansible/playbooks/openstack/inventory.py -i inventory /usr/share/ansible/openshift-ansible/playbooks/openstack/openshift-cluster/provision_install.yml
```

Assuming you used the OpenStack Keypair from your current user on your ansible host, you can access instances via openshift user:
```
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# ssh openshift@192.168.1.81
```

## Modifying the playbooks to use IPA Ansible Modules instead of nsupdate

In the repo, there is a modified populate-dns.yml to use IPA modules instead of nsupdate. Copy this to /usr/share/ansible/openshift-ansible to replace the existing file.

```
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# cp /usr/share/ansible/openshift-ansible/roles/openshift_openstack/tasks/populate-dns.yml /root/
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# cp ./populate-dns.yml /usr/share/ansible/openshift-ansible/roles/openshift_openstack/tasks/
```

What the file looks like after mods
```
(ocp-on-rhosp) (overcloud) [root@ansible-host ocp-on-rhosp]# cat /usr/share/ansible/openshift-ansible/roles/openshift_openstack/tasks/populate-dns.yml
---
- name: Add private node records
  ipa_host:
    fqdn: "{{ (hostvars[item]['ansible_hostname'] + openshift_openstack_private_hostname_suffix + '.' + openshift_openstack_full_dns_domain) }}"
    description: "{{ (hostvars[item]['ansible_hostname'] + openshift_openstack_private_hostname_suffix + '.' + openshift_openstack_full_dns_domain) }}"
    ip_address: "{{ hostvars[item]['private_v4'] }}"
    ns_host_location: openshift-cluster
    ns_os_version: RHEL 7
    ns_hardware_platform: OpenStack Instance
    state: "{{ l_dns_record_state | default('present') }}"
    ipa_host: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
    ipa_user: "{{ openshift_openstack_external_nsupdate_keys['public']['username'] }}"
    ipa_pass: "{{ openshift_openstack_external_nsupdate_keys['public']['password'] }}"
    update_dns: yes
    validate_certs: false
  with_items: "{{ l_openshift_openstack_dns_update_nodes }}"
  when:
    - openshift_openstack_external_nsupdate_keys['private'] is defined
    - hostvars[item]['private_v4'] is defined
    - hostvars[item]['private_v4'] is not none
    - hostvars[item]['private_v4'] | string

- name: Add public wildcard record
  ipa_dnsrecord:
    ipa_host: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
    ipa_user: "{{ openshift_openstack_external_nsupdate_keys['public']['username'] }}"
    ipa_pass: "{{ openshift_openstack_external_nsupdate_keys['public']['password'] }}"
    state: "{{ l_dns_record_state | default('present') }}"
    zone_name: "{{ openshift_openstack_nsupdate_zone }}"
    record_name: "{{ ('*.' + hostvars[groups.masters[0]].openshift_master_default_subdomain) | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
    record_type: "A"
    record_value: "{{ hostvars[groups.lb[0]]['public_v4'] }}"
    #record_ttl: 300 #NEEDED only if Ansible > 2.7
    validate_certs: false
  when:
    - openshift_openstack_external_nsupdate_keys['public'] is defined
    - groups.lb
    - hostvars[groups.masters[0]].openshift_master_default_subdomain is defined
#    - openshift_openstack_public_router_ip is defined
#    - openshift_openstack_public_router_ip is not none
#    - openshift_openstack_public_router_ip | string

- name: Add public API record
  ipa_dnsrecord:
    ipa_host: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
    ipa_user: "{{ openshift_openstack_external_nsupdate_keys['public']['username'] }}"
    ipa_pass: "{{ openshift_openstack_external_nsupdate_keys['public']['password'] }}"
    state: "{{ l_dns_record_state | default('present') }}"
    zone_name: "{{ openshift_openstack_nsupdate_zone }}"
    record_name: "{{ hostvars[groups.masters[0]].openshift_master_cluster_public_hostname | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
    record_type: "A"
    record_value: "{{ openshift_openstack_public_api_ip }}"
    #record_ttl: 300 #NEEDED for Ansible > 2.7
    validate_certs: false
  when:
    - openshift_openstack_external_nsupdate_keys['public'] is defined
    - groups.masters
    - hostvars[groups.masters[0]].openshift_master_cluster_public_hostname is defined
    - openshift_openstack_public_api_ip is defined
    - openshift_openstack_public_api_ip is not none
    - openshift_openstack_public_api_ip | string
```
