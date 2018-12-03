sshd_apply
==========

Setup SSH service from a template. A rollback feature ensures you will not be
locked out the target host.

Requirements
------------

None.

Role Variables
--------------

This role can be set using the following variables:

* Port to switch on to confirm applied/current/runtime configuration. May be
  `yes` (try to guess port number from new configuration), `no`, or a port
  number.

```yaml
sshd_apply__switch_port: yes
```

* Path of a template file that once evaluated is used as configuration to be
  loaded by sshd.

```yaml
sshd_apply__template: sshd_apply.j2
```

* The delay, in seconds, after what the initial ssh server's configuration is
  reloaded if the confirmation file is missing.

```yaml
sshd_apply__timeout: 20
```

Template Variables
------------------

The template provided with this role also supports `sshd_config`'s options as
variable names with `sshd__` as prefix, followed by the name of the option, in
lowercase. For example:

```yaml
sshd__port: 2222
sshd__usepam: yes
sshd__permitrootlogin: forced-commands-only
sshd__passwordauthentication: no
```

will give:

```
Port 2222
UsePAM yes
PermitRootLogin forced-commands-only
PasswordAuthentication: no
```

### BOOLEANS

Even if ssh booleans only accept 'yes' or 'no' values, the template accepts
all ansible's booleans and translates them to 'yes' or 'no'. **NOTE** that it
is a feature of the template provided by the role, *not a feature of the role
 itself*. If you point `sshd_apply__template` to your own, ensure to do the
same, otherwise something like:

```yaml
sshd__passwordauthentication: no
```
would end up in your `sshd_config` as:
```yaml
PasswordAuthentication False
```
and fail.

This translation to `yes`/`no` values also applies to mixed-type options that
accept either a boolean or a keyword, as do `PermitRootLogin` or `PermitTunnel`

### LISTS

Some `sshd_config`'s options may accept a comma-separated or a space-separated
list of values in a single call. For example:

```
AllowUsers alice bob
Ciphers chacha20-poly1305@openssh.com,aes256-ctr,aes256-gcm@openssh.com
```

although other options have to be specified as many times as needed, such as:

```
Port 22
Port 2222
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key
```

For first options, template variables accept both a string that will be passed
as is to the option, or a yaml list that will be processed to be passed to the
option as a comma-separated or space-separated list, depending on the option.

For second options, template variables accept both a string that will be passed
as is to the option, or a yaml list that will be processed to be passed to a
list of the same option, one value per call of the option. For example:

* These are correct declarations:

```yaml
sshd__port: 2222
sshd__port: [ 2222 ]
sshd__port:
  - 2222
sshd__port: [ 22, 2222 ]
sshd__port:
  - 22
  - 2222
```

* These are incorrect declarations:

```yaml
sshd__port: 22 2222
sshd__port: 22,2222
```


Dependencies
------------

None.

Example Playbook
----------------

```yaml
- hosts: servers
  roles:
    - role: sshd_apply
      sshd__passwordauthentication: no
      sshd__permitrootlogin: forced-commands-only
      sshd__usepam: yes
      sshd__acceptenv:
        - LANG
	- LC_*
```

License
-------

GPLv3

Author Information
------------------

<quidame@poivron.org>