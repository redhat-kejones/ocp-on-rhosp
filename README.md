## Ansible Host Setup

You need to have a host for which to run the playbooks from. Assuming a RHEL 7.5+ box

Configure the proper repos and install the needed packages

```
[root@ansible-host ~]# subscription-manager repos --disable=* --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-7-server-ansible-2.6-rpms --enable=rhel-7-server-ose-3.11-rpms --enable=rhel-7-server-openstack-14-rpms

[root@ansible-host ~]# yum install -y git ansible python-openstackclient python-heatclient openshift-ansible python2-shade python-dns python-virtualenv
```

Clone this repo so we have a baseline to operate from
```
[root@ansible-host ~]# git clone https://gitlab.consulting.redhat.com/kejones/orock-ocp-on-rhosp.git
[root@ansible-host ~]# cd orock-ocp-on-rhosp
```

In order to control what libraries get installed and their dependencies, I'll use virtualenv
```
[root@ansible-host orock-ocp-on-rhosp]# virtualenv ./
[root@ansible-host orock-ocp-on-rhosp]# . bin/activate
(orock-ocp-on-rhosp) [root@ansible-host orock-ocp-on-rhosp]# pip install shade
(orock-ocp-on-rhosp) [root@ansible-host orock-ocp-on-rhosp]# pip install dnspython
(orock-ocp-on-rhosp) [root@ansible-host orock-ocp-on-rhosp]# pip install jinja2
```

Copy a few pieces from the openshift-ansible project
```
(orock-ocp-on-rhosp) [root@ansible-host orock-ocp-on-rhosp]# cp -a /usr/share/ansible/openshift-ansible/playbooks/openstack/sample-inventory ./inventory
(orock-ocp-on-rhosp) [root@ansible-host orock-ocp-on-rhosp]# cp /usr/share/ansible/openshift-ansible/playbooks/openstack/ansible.cfg ./
```

Add any_errors_fatal setting to ansible.cfg [defaults] section
```
(orock-ocp-on-rhosp) [root@ansible-host orock-ocp-on-rhosp]# vi ansible.cfg 
...
[defaults]
...
any_errors_fatal = true
...
```

Make sure that you have an OpenStack rc file to auth to your OpenStack cloud. I got my operatorrc file from my undercloud. This authenticates me as operator user on my RHOSP 14 overcloud
```
(orock-ocp-on-rhosp) [root@ansible-host orock-ocp-on-rhosp]# scp stack@192.168.0.5:operatorrc ./
(orock-ocp-on-rhosp) [root@ansible-host orock-ocp-on-rhosp]# source operatorrc
```

Now I should be able to run openstack commands and get responses
```
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# openstack server list
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# openstack image list
```

You need to now edit your all group_vars file
```
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# vi inventory/group_vars/all.yml
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# vi inventory/group_vars/OSEv3.yml
```

## Running openshift-ansible OpenStack playbooks

Now you are ready to run the playbooks.

If you just want to run the provision piece for OpenStack Heat stack creation
```
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# ansible-playbook -i /usr/share/ansible/openshift-ansible/playbooks/openstack/inventory.py -i inventory /usr/share/ansible/openshift-ansible/playbooks/openstack/openshift-cluster/provision.yml
```

> NOTE: in between provision and installation, you should set up all forward and reverse DNS records
>       if you are not using the mod later in this README to do IPA/IdM DNS population

After heat stack creation and DNS setup, you can run the install piece for OpenShift installation
```
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# ansible-playbook -i /usr/share/ansible/openshift-ansible/playbooks/openstack/inventory.py -i inventory /usr/share/ansible/openshift-ansible/playbooks/openstack/openshift-cluster/install.yml
```

If you want to run it all in one go, do the following
```
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# ansible-playbook -i /usr/share/ansible/openshift-ansible/playbooks/openstack/inventory.py -i inventory /usr/share/ansible/openshift-ansible/playbooks/openstack/openshift-cluster/provision_install.yml
```

Assuming you used the OpenStack Keypair from your current user on your ansible host, you can access instances via openshift user:
```
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# ssh openshift@192.168.1.81
```

## Modifying the playbooks to use IPA Ansible Modules instead of nsupdate

In the repo, there is a modified populate-dns.yml to use IPA modules instead of nsupdate. Copy this to /usr/share/ansible/openshift-ansible to replace the existing file.

```
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# cp /usr/share/ansible/openshift-ansible/roles/openshift_openstack/tasks/populate-dns.yml /root/
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# cp ./populate-dns.yml /usr/share/ansible/openshift-ansible/roles/openshift_openstack/tasks/
```

What the file looks like after mods
```
(orock-ocp-on-rhosp) (overcloud) [root@ansible-host orock-ocp-on-rhosp]# cat /usr/share/ansible/openshift-ansible/roles/openshift_openstack/tasks/populate-dns.yml
---
- name: Add private node records
  ipa_dnsrecord:
    ipa_host: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
    ipa_user: "{{ openshift_openstack_external_nsupdate_keys['public']['username'] }}"
    ipa_pass: "{{ openshift_openstack_external_nsupdate_keys['public']['password'] }}"
    state: present
    zone_name: "{{ openshift_openstack_nsupdate_zone }}"
    record_name: "{{ (hostvars[item]['ansible_hostname'] + openshift_openstack_private_hostname_suffix + '.' + openshift_openstack_full_dns_domain) | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
    record_type: "A"
    record_value: "{{ hostvars[item]['private_v4'] }}"
    #record_ttl: 300 #NEEDED only if Ansible > 2.7
    validate_certs: false
#  nsupdate:
#    key_name: "{{ openshift_openstack_external_nsupdate_keys['private']['key_name'] }}"
#    key_secret: "{{ openshift_openstack_external_nsupdate_keys['private']['key_secret'] }}"
#    key_algorithm: "{{ openshift_openstack_external_nsupdate_keys['private']['key_algorithm'] | lower }}"
#    server: "{{ openshift_openstack_external_nsupdate_keys['private']['server'] }}"
#    zone: "{{ openshift_openstack_nsupdate_zone }}"
#    record: "{{ (hostvars[item]['ansible_hostname'] + openshift_openstack_private_hostname_suffix + '.' + openshift_openstack_full_dns_domain) | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
#    value: "{{ hostvars[item]['private_v4'] }}"
#    type: "A"
#    state: "{{ l_dns_record_state | default('present') }}"
  with_items: "{{ l_openshift_openstack_dns_update_nodes }}"
#  register: nsupdate_add_result
#  until: nsupdate_add_result is succeeded
#  retries: 10
  when:
    - openshift_openstack_external_nsupdate_keys['private'] is defined
    - hostvars[item]['private_v4'] is defined
    - hostvars[item]['private_v4'] is not none
    - hostvars[item]['private_v4'] | string
#  delay: 1


- name: Add public node records
  ipa_dnsrecord:
    ipa_host: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
    ipa_user: "{{ openshift_openstack_external_nsupdate_keys['public']['username'] }}"
    ipa_pass: "{{ openshift_openstack_external_nsupdate_keys['public']['password'] }}"
    state: present
    zone_name: "{{ openshift_openstack_nsupdate_zone }}"
    record_name: "{{ (hostvars[item]['ansible_hostname'] + openshift_openstack_public_hostname_suffix + '.' + openshift_openstack_full_dns_domain) | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
    record_type: "A"
    record_value: "{{ hostvars[item]['public_v4'] }}"
    #record_ttl: 300 #NEEDED only if Ansible > 2.7
    validate_certs: false
#  nsupdate:
#    key_name: "{{ openshift_openstack_external_nsupdate_keys['public']['key_name'] }}"
#    key_secret: "{{ openshift_openstack_external_nsupdate_keys['public']['key_secret'] }}"
#    key_algorithm: "{{ openshift_openstack_external_nsupdate_keys['public']['key_algorithm'] | lower }}"
#    server: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
#    zone: "{{ openshift_openstack_nsupdate_zone }}"
#    record: "{{ (hostvars[item]['ansible_hostname'] + openshift_openstack_public_hostname_suffix + '.' + openshift_openstack_full_dns_domain) | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
#    value: "{{ hostvars[item]['public_v4'] }}"
#    type: "A"
#    state: "{{ l_dns_record_state | default('present') }}"
  with_items: "{{ l_openshift_openstack_dns_update_nodes }}"
#  register: nsupdate_add_result
#  until: nsupdate_add_result is succeeded
#  retries: 10
  when:
    - openshift_openstack_external_nsupdate_keys['public'] is defined
    - hostvars[item]['public_v4'] is defined
    - hostvars[item]['public_v4'] is not none
    - hostvars[item]['public_v4'] | string
#  delay: 1

- name: Add public wildcard record
  ipa_dnsrecord:
    ipa_host: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
    ipa_user: "{{ openshift_openstack_external_nsupdate_keys['public']['username'] }}"
    ipa_pass: "{{ openshift_openstack_external_nsupdate_keys['public']['password'] }}"
    state: present
    zone_name: "{{ openshift_openstack_nsupdate_zone }}"
    record_name: "{{ ('*.' + hostvars[groups.masters[0]].openshift_master_default_subdomain) | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
    record_type: "A"
    record_value: "{{ openshift_openstack_public_router_ip }}"
    #record_ttl: 300 #NEEDED only if Ansible > 2.7
    validate_certs: false
#  nsupdate:
#    key_name: "{{ openshift_openstack_external_nsupdate_keys['public']['key_name'] }}"
#    key_secret: "{{ openshift_openstack_external_nsupdate_keys['public']['key_secret'] }}"
#    key_algorithm: "{{ openshift_openstack_external_nsupdate_keys['public']['key_algorithm'] | lower }}"
#    server: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
#    zone: "{{ openshift_openstack_nsupdate_zone }}"
#    record: "{{ ('*.' + hostvars[groups.masters[0]].openshift_master_default_subdomain) | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
#    value: "{{ openshift_openstack_public_router_ip }}"
#    type: "A"
#    state: "{{ l_dns_record_state | default('present') }}"
#  register: nsupdate_add_result
#  until: nsupdate_add_result is succeeded
#  retries: 10
#  delay: 1
  when:
    - openshift_openstack_external_nsupdate_keys['public'] is defined
    - groups.masters
    - hostvars[groups.masters[0]].openshift_master_default_subdomain is defined
    - openshift_openstack_public_router_ip is defined
    - openshift_openstack_public_router_ip is not none
    - openshift_openstack_public_router_ip | string


- name: Add public API record
  ipa_dnsrecord:
    ipa_host: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
    ipa_user: "{{ openshift_openstack_external_nsupdate_keys['public']['username'] }}"
    ipa_pass: "{{ openshift_openstack_external_nsupdate_keys['public']['password'] }}"
    state: present
    zone_name: "{{ openshift_openstack_nsupdate_zone }}"
    record_name: "{{ hostvars[groups.masters[0]].openshift_master_cluster_public_hostname | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
    record_type: "A"
    record_value: "{{ openshift_openstack_public_api_ip }}"
    #record_ttl: 300 #NEEDED for Ansible > 2.7
    validate_certs: false
#  nsupdate:
#    key_name: "{{ openshift_openstack_external_nsupdate_keys['public']['key_name'] }}"
#    key_secret: "{{ openshift_openstack_external_nsupdate_keys['public']['key_secret'] }}"
#    key_algorithm: "{{ openshift_openstack_external_nsupdate_keys['public']['key_algorithm'] | lower }}"
#    server: "{{ openshift_openstack_external_nsupdate_keys['public']['server'] }}"
#    zone: "{{ openshift_openstack_nsupdate_zone }}"
#    record: "{{ hostvars[groups.masters[0]].openshift_master_cluster_public_hostname | replace('.' + openshift_openstack_nsupdate_zone, '') }}"
#    value: "{{ openshift_openstack_public_api_ip }}"
#    type: "A"
#    state: "{{ l_dns_record_state | default('present') }}"
#  register: nsupdate_add_result
#  until: nsupdate_add_result is succeeded
#  retries: 10
#  delay: 1
  when:
    - openshift_openstack_external_nsupdate_keys['public'] is defined
    - groups.masters
    - hostvars[groups.masters[0]].openshift_master_cluster_public_hostname is defined
    - openshift_openstack_public_api_ip is defined
    - openshift_openstack_public_api_ip is not none
    - openshift_openstack_public_api_ip | string
```
