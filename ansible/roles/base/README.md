Base
=========

This role will prepare the kragle autodeploynode with the all the resources needed to deploy Red Hat OpenStack on RHEL. It will install the necessary packages, configure them and clone the required git repositories.

Requirements
------------

The role requires a pre-installed version of RHEL with internet connectivity performed from the instructions and kickstart files provided in this repository.

Role Variables
--------------

The variables for this role come from facts gathered by ansible and vars/main.yml. The vars/main.yml variables need to be edited as needed.

Dependencies
------------

The role depends on the requirements listed above.

Playbook
--------

The role is included in the main.yml in kragle/ansible
From the root directory of the kragle repo

    ansible-playbook ansible/main.yml
