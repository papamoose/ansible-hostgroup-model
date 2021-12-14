# Ansible Hostgroup Model

I've been spending a lot more time with Ansible lately I've been unhappy with role organization, how variables get applied, and that groups or hosts cannot assert roles that should be applied to them.

My background is using Puppet in production for 8+ years. Though, I've also used Ansible for a similar time, I've never needed to think about managing more than 30 or machines. Of those I was applying all roles to all machines.

My current deep think has resulted in what I think is a fairly successful attempt at recreating a hierarchical inheritance model, similar to Puppet's hostgroups, which allows a host or a group to define which roles should apply to it, rather than the playbook.


## Role organization

Organize your roles into multiple directories.

At the moment I use three folders. `./roles`, `./organizations`, and `./hostgroups`.

Where you put or what you name these directories is up to you, but you'll want to modify `roles_path` in `ansible.cfg` to include all directories.

```
roles_path = ./hostgroups:./roles:./organization
```

`./roles`: for any role that will be included or imported in some way from another role, task, play, etc.

`./organization`: host or cluster specific roles. Roles you don't intend on using anywhere else.

`./hostgroups`: This is how we setup a default hierarchy for roles that get applied to hosts. You won't be adding or removing from this directory after you've setup a hierarchy.

Now you can create a hierarchical structure with the following inheritance pattern: `base > server > headless > desktop`

`./hostgroups/base`: Contains items that all hosts should have installed

`./hostgroups/server`: Includes `base` and contains all roles that should be applied to servers.

`./hostgroups/headless`: Includes `server` and contains all user land tools researchers could want.

`./hostgroups/desktop`: Includes `headless` and contains WMs,LM,DM, etc...


## Create a role to include_role based on the hostgroup specified

Since my goal is allow a host or group of hosts to define the roles that should be applied to them I needed a way to change the default model; which is to develop a play which has a list of roles applied to it. However, I found that this always ended up with a long and messy playbook file or many different playbook files, where some hosts might be listed more than once. If I have hundreds of machines all with slightly different role sets and the only way to to apply a role to a machine is via a playbook then playbooks could be thousands of lines long.

In order to give a host or group the power to specify any roles that should apply to them I've created a simple role which iterates over a list of roles you specify.

This role requires that the variable `hostgroup` has been set to one of the hostgroups created above.

e.g. `hostgroup: headless`

`roles/hostgroup-include-roles/tasks/main.yml`
```
---
- name: include a role from a list of 'roles'
  ansible.builtin.include_role:
    name: "{{ hostgroup | default('base') }}"
    allow_duplicates: yes
  tags:
  - always
```

`site.yml`
```
---
- hosts: all
  user: deployuser
  become: yes
  gather_facts: yes
  roles:
  - hostgroup-include-roles  # first role in the playbook
```


### Setting up hostgroup inheritance

Inside `./hostgroups` we create the inheritance structure we need to import roles

First we setup the structure inside `./group_vars/all/roles.yml`. The are lists that will control which roles will be applied to a hostgroup.

```
roles_base:
- ssh
- apt

roles_server:
- iptables

roles_headless:
- etc-hosts

roles_desktop:
- lightdm
- ubuntu-desktop
```

We create a task that combines the lists `roles_*` and include all of the listed roles in hierarchical order. There also exists hostgroup roles for `base`,`server`, and `headless` but only include the necessary lists of roles.

For example: In the case of `hostgroup=desktop`

`hostgroups/desktop/tasks/main.yml`
```
- name: include a role from a list of 'roles'
  ansible.builtin.include_role:
    name: "{{ r }}"
    allow_duplicates: no
  loop: "{{ (roles_base + roles_server + roles_headless + roles_desktop + roles_group + roles_host) | list }}"
  loop_control:
    loop_var: r
  tags:
  - always
```

`roles_group:[]`: Apply roles to a group of machines. Should **only** be used inside group_vars.

`roles_host:[]`: Apply roles to a host. Should **only** be used inside host_vars.

Because we maintain the hierarchical structure it means if you create a role for a `headless` system you should be able to depend on the configurations from `base` and `server` being there.


## Demo

Note that you can also apply a role directly to the host. I've set the following in `host_vars` to append roles to a host.

`host_vars/localhost/roles.yml`
```
---
roles_host:
- var-override
```

### Example 1

Using the default hostgroup role group in the inventory file.
`inventory/example1`
```
[server]
localhost
```

The following roles should apply: `ssh`, `apt`, `iptables`, `vars-override`.
```
$ ansible-playbook -i inventory/example1 site.yml 

PLAY [all] ********************************************************************************

TASK [rm-example-dir : file] **************************************************************
changed: [localhost]

TASK [rm-example-dir : organization base | directory] *************************************
changed: [localhost]

TASK [include a role from a list of 'roles'] **********************************************

TASK [server : debug] *********************************************************************
ok: [localhost] => {
    "msg": "Applying hostgroup server"
}

TASK [include a role from a list of 'roles'] **********************************************

TASK [ssh : debug] ************************************************************************
ok: [localhost] => {
    "msg": "role ssh"
}

TASK [apt : debug] ************************************************************************
ok: [localhost] => {
    "msg": "role apt"
}

TASK [iptables : debug] *******************************************************************
ok: [localhost] => {
    "msg": "role iptables"
}

TASK [var-override : var-override test | create file] *************************************
changed: [localhost]

PLAY RECAP ********************************************************************************
localhost                  : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```


### Example 2

This example represents a set of related systems which should be configured a little differently.

`agents` are nodes which users will log into and thus require a standard set of tools and packages to be installed.

The `control` is not for users and only needs the basic set of packages and configurations determined to be part of the server hostgroup.

`inventory/example2`
```
[control]
node1

[agents]
localhost
node2
node3

[control:vars]
hostgroup=server
roles_group

[agents:vars]
hostgroup=headless
```


The following roles should apply: `ssh`, `apt`, `iptables`, `etc-hosts` `vars-override`.

Since the demo only includes configuring localhost I limit this run.

```
$ ansible-playbook -i inventory/example2 site.yml -l localhost

PLAY [all] *****************************************************************************************************************************

TASK [rm-example-dir : file] ***********************************************************************************************************
changed: [localhost]

TASK [rm-example-dir : organization base | directory] **********************************************************************************
changed: [localhost]

TASK [include a role from a list of 'roles'] *******************************************************************************************

TASK [headless : debug] ****************************************************************************************************************
ok: [localhost] => {
    "msg": "Applying hostgroup server"
}

TASK [include a role from a list of 'roles'] *******************************************************************************************

TASK [ssh : debug] *********************************************************************************************************************
ok: [localhost] => {
    "msg": "role ssh"
}

TASK [apt : debug] *********************************************************************************************************************
ok: [localhost] => {
    "msg": "role apt"
}

TASK [iptables : debug] ****************************************************************************************************************
ok: [localhost] => {
    "msg": "role iptables"
}

TASK [etc-hosts : Manage /etc/hosts] ***************************************************************************************************
changed: [localhost]

TASK [var-override : var-override test | create file] **********************************************************************************
changed: [localhost]

PLAY RECAP *****************************************************************************************************************************
localhost                  : ok=8    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## Thoughts. Not Final.

After this investigation it has become clear to me that Ansible core has a very specific usage model and if you stray outside of that you won't be able to necessarily bend it to your will and if you do, risk that many roles stop working as intended.

My solution of importing roles based on variable names works like I intended, in that it is not necessary to define all roles that should be applied to a system. Only the extra roles you'd like to apply on top of what the hostgroup provides.

Not using `hash_behaviour=merge` prevents me from auto merging lists and dictionaries. In my testing I had a working solution which merged a lists of roles with the same name and applied those to the hosts. However, because the order of the application of those roles could not be guaranteed I thought that was just asking for trouble. Especially because Ansible doesn't have a notion that any role could run at any time in any order like Puppet. You must specify all of your dependencies and ordering in the module.



## Misc and related items

### hash_behaviour=merge

During my research I attempted switching to `hash_behaviour=merge` with success until I found out it was deprecated and on the chopping block to be removed. Someone was able to get the devs to not remove it last minute but the devs expressed a severe dislike to `merge` that it would be unwise to try to use it. I think it would be fine if you write all of the ansible roles/libraries/plugins etc yourself, or at least had the capability to vendor and make them work under `merge`. Honestly, I didn't see any argument that has convinced me not to use merge besides the developers hate for it.

[Chat link](https://meetbot.fedoraproject.org/ansible-meeting/2021-01-21/ansible_core_public_irc_meeting.2021-01-21-15.01.log.html)

> 15:25:47 <agaffney> the problem is that it's a global feature that can wreak havoc on third party code that isn't designed for it

> 16:12:24 <agaffney> Duck: because VarsManager is already a mostly unmaintainable mess and adding more complexity to it is a terrible idea

> 16:13:44 <bcoca> sivel: im want to deprecate it, but i want to remove many things that im not able to (debug, roles, dependencies, varsmanager, etc)

I was left with a bad taste of the Ansible project after reading this chat. I assume <bcoca> has good reasons to remove the items he lists, but without context the ideas seem extreme.

They also all seem to advocate the replication of data into various roles, host_vars, and group_vars; and structures in Ansible that would necessitate mixing code and data more than I think is reasonable.

This was a deep rabbit hole, which I ultimately was able to reject by changing my techniques, but I thought it was still worth including this section as a "lessons learned" section. 


## Inventory Sources

One of which is The Foreman which seems to also export hostgroups to be used as Ansible groups. In effect, doing the work I did above.

* [Inventory Source Index](https://docs.ansible.com/ansible/latest/collections/index_inventory.html)
* [The Foreman](https://docs.ansible.com/ansible/latest/collections/theforeman/foreman/foreman_inventory.html#ansible-collections-theforeman-foreman-foreman-inventory)
  * `want_host_group`: Toggle, if true the inventory will fetch host_groups and create groupings for the same.
  * `want_hostcollections`: Toggle, if true the plugin will create Ansible groups for host collections

### The Future

The Foreman is probably necessary to be used as the inventory system. It can generate the appropriate kickstart and preseed files for a host, provide variables based on the IP addresses you assign a host, and has the ability to export hostgroups and a bunch of information about the machine you configure at build time to Ansible.

The Foreman works best when you tie it with your DHCP and TFTP servers so work would have to be done to make the current setup compatible with 'theforeman-proxy' that would have to be run on those machines.
