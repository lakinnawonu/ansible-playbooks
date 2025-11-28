Role Name
satellite-build

Requirements
------------
This role installs the satellite server on a suitable system that meets the installation requirement. This role expects that the server has a minimum of 30GB of ram, 400GB of unused/unpartitioned disk space and alteast 4CPU cores. The hostname of the server should also be set ( hostnamectl set-hostname )

minimum ram: 20GB
minimum disk size: 400GB
minimu CPU: 4-core 2.0 GHz
A unique host name



Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

- name: Install satellite server
  hosts: satellite-server
  vars:
    ansible_ssh_pipelining: true
  roles:
    - satellite-build

License
-------

BSD

Author Information
------------------
Author: Larry Akinnawonu

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
