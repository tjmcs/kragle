Bridge
=========

This role creates a bridge on the physical interface for use with KVM.

Requirements
------------

The ifcfg_br0.j2 template has variables found in the vars/main.yml.

Role Variables
--------------

The variables for this role come from facts gathered by ansible and vars/main.yml. The vars/main.yml variables need to be edited as needed.

Dependencies
------------

This role will get applied with the base or kragle-packer roles and is included in the main.yml and main-with-packer.yml plays.
