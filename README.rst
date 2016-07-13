ansible-role-vmiterate
==============================================================================

Ansible role to iteratively destroy, create, and provision a
development VM using thin-LV snapshots of vendor-provided cloud
images.  The VMs bootstrap themselves using `cloud-init` with
templatized metadata we provide, ready for Ansible to complete
configuration in a separate role.

Facilitates rapid edit/build/test cycle at system level, and
allows for easy migration from local VMs to system containers or
to cloud VMs: they would all use the same images, bootstrap
method, and configuration tool.

Tested using CentOS and Ubuntu vendor cloud images.

| scott@smemsh.net
| http://smemsh.net/src/ansible-role-vmiterate
| http://spdx.org/licenses/AGPL-3.0


result
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Upon role completion, the VM is ready to boot, and will
provision itself using a `cloud-init` template the role writes
onto a small configuration drive created and installed into the
VM (created and added to the libvirt "domain XML" which we also
template).

A configuration management user is created in the VM, and made
available to the host for login via ssh on the configured
virtual network.  The VM is now ready for the CM tool to
complete system configuration, in a separate role.


preparation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The cloud image must have been written beforehand (using eg
`qemu-img -O raw`) onto an LVM thin volume which the role will
snapshot (and make anew on each re-run) for each VM iteration.

Any changes to the base image that should be in each iteration
(and cannot be specified in the role's `meta-data` or
`user-data` templates) should be put into the origin LV.

This role as written uses static IPs (subnet and host offsets
are configured via ansible variables), so we disable DHCP in our
cloud images.  The root filesystem backing device is also
expanded manually in the origin images used to test this role,
but no other changes are needed for vendor-provided cloud
images.


configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The role uses several variables for system and site-specific
details, which need to be passed into the role via playbook, or
configured in group/host variables.

#. edit `vars/config.yml`_
#. define the following variables (presumably via host/group vars):

       :domain: dns domain for the VMs
       :hostkey_md5: md5 fingerprint of rsa hostkey (lower, no colons)
       :ssh_auth_md5: md5 fingerprint of sysadmin pubkey (same)
       :ssh_auth_pubstrs[]: dict of sysadmin public key text, keyed by md5 fp
       :ssh_host_keystrs[]: dict of private key text, keyed by md5 fp
       :ssh_host_pubstrs[]: dict of private key text, keyed by md5 fp
       :ip_offset: ip offset into the subnet for the vm guest address
       :lvbase: variable part of origin lvname (`config.yml` for static parts)
       :secret_cfmgmt_password: password for cm user (fill from vault/crypt)

   see http://smemsh.net/src/.ansible/group_vars/smemsh for a
   clever way to provide the keys using files with the name of
   the host and its md5 via `lookup('file')` in the playbook
   `files/`, or just add them manually

   see http://smemsh.net/src/.ansible/host_vars/ubuplex for an
   example per-guest hostvars

.. _vars/config.yml: vars/config.yml


procedure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following steps are taken by the role during its run:


destroy
-------

Removes the previous vm instance, if it exists, along with any
temporary mounts of its root filesystem the developer has used
to inspect or modify the guest for diagnosing problems or
testing her changes.


create
------

Instantiate a new instance, in this case a libvirt-managed
qemu-kvm for which we have templatized the xml for in this role.
This snapshots the origin LV and defines the VM using libvirt.


provision
---------

Create a `cloud-init` configuration drive (vfat filesystem with
`cidata` label, added to the libvirt domain for the guest VM),
that gets `meta-data` and `user-data` that will cause the
following provisioning events to occur once the VM gets booted:

    - static network
    - system locale
    - rsa hostkey
    - config superuser
    - ssh authkey

Configuration inside the VM is left to a separate role

invocation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The role needs to have the 'host' variable passed to it as an
extra arg on the command line.  This serves as the name of the
VM as known by libvirt, gets configured as the hostname, and is
used to take several variables out of hostvars[host] (see
`configuration`_)


status
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- used by author to instantiate vms for testing devops
- please notify author if using
- some aspects specific to author environment (edit vars/config.yml)


todo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- split create and destroy (tags?)
- provide and link to configuration roles
- followup roles that configure base packages, create user, do other stuff
- gce, lxd, linode, ec2
- no reason to expand fs manually since cloud-init can resize it
- provide example `wget`, `guestfish`, `lvm` commands to get/write image
- move some possible site-local from config.yml into host/group vars
- use defaults.yml so config.yml can be used for overrides only?
- reduce variables needing configuration and/or expect them in role args
