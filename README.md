sshd_apply
==========

Setup OpenSSH service from a template. A rollback feature ensures the Ansible
Controller will not be locked out the target host.

**SUMMARY**

- [Description](#description)
- [Requirements](#requirements)
- [Role Variables](#role-variables)
- [Template Variables](#template-variables)
  - [booleans](#booleans)
  - [lists](#lists)
  - [objects](#objects)
- [Dependencies](#dependencies)
- [Example Playbook](#example-playbook)
- [Galaxy](#galaxy)
- [License](#license)
- [Author Information](#author-information)


Description
-----------

This role populates target's `sshd_config` from a template, and reloads ssh
daemon with this new configuration. If the next task fails, meaning that the
target is not reachable anymore, the daemon is restarted with its initial
configuration, so the ansible controller, at least, is not locked out of its
target.

When reconfiguring SSH remotely, you'll have to restart it. Common cases of
lock out may appear when you want to harden the service:

* the `Port` value has changed and
  - the firewall is not aware of that and the new port is blocked;
  - the new port is already in use and SSH died at reload/restart;
* the `ansible_user` is missing in the new list `AllowUsers`;
* a `Match` directive with restrictive options unfortunately applies to the
  `ansible_user`;
* ...

Some of these cases may still apply to this role, except that the lock out is
temporary (defaults to 20 seconds). As the `sshd_config` is not overwritten
**before** reloading the daemon, even if the rollback fails (that is a very
bad situation and shouldn't happen), rebooting the host will restore its
initial ssh configuration.

The template provided by this role exhaustively supports all options of the
server. See the [Template Variables](#template-variables) section below.

Requirements
------------

None.

Role Variables
--------------

This role can be set using the following variables:

* Whether or not to make currently applied configuration persistent. If
  `False`, the sshd configuration set by the role will still apply after the
  role has been played, but this configuration will not be reapplied after
  reboot.

```yaml
sshd_apply__save_state: yes
```

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

The template provided with this role supports all `sshd_config`'s options as
variable names with `sshd__` as prefix followed by the name of the option, in
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
PasswordAuthentication no
```

### booleans

Even if ssh booleans only accept `yes` or `no` values, the template accepts
all ansible's booleans and translates them to `yes` or `no`. **NOTE** that it
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
accept either a boolean or a keyword, as do `PermitRootLogin` or `PermitTunnel`.

### lists

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

For first options (`AllowUsers`, `Ciphers` ...) template variables accept both
a string that will be passed as is to the option, or a yaml list that will be
processed to be passed to the option as a comma-separated or space-separated
list, depending on the option.

For second options (`Port`, `ListenAddress` ...) template variables accept both
a string that will be passed as is to the option, or a yaml list that will be
processed to be passed to a list of the same option, one value per line.

For example:

* These are correct declarations:

```yaml
sshd__allowusers: alice bob
sshd__allowusers:
  - alice
  - bob
sshd__ciphers: chacha20-poly1305@openssh.com,aes256-ctr,aes256-gcm@openssh.com
sshd__ciphers:
  - chacha20-poly1305@openssh.com
  - aes256-ctr
  - aes256-gcm@openssh.com
```

* These are incorrect declarations:

```yaml
sshd__allowusers: alice,bob
sshd__ciphers: chacha20-poly1305@openssh.com aes256-ctr aes256-gcm@openssh.com
```

* These are correct declarations:

```yaml
sshd__port: 2222
sshd__port: [ 2222 ]
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

If you are in doubt about what inline list format is expected (spaces or commas)
or if it is a multi-decarative option instead, just use a yaml list.

### objects

As a `Match` directive:

- accepts two parameters, a criteria and a series of patterns matching this
  criteria;
- applies a subset of global options to objects caught by the matching rule;
- may appear more than once;

the `sshd__match` is a complex object whose values are passed verbatim by the
template to `sshd_config`.

```yaml
sshd__match:
  - criteria: CRITERIA
    patterns: PATTERN1,PATTERN2
    options:
      - "OptionOne value1"
      - "OptionTwo value2"
      - "OptionSix value6"
  - criteria: CRITERIA
    patterns:
      - PATTERNx
      - PATTERNy
      - PATTERNz
    options:
      - "OptionFour value4"
      - "OptionFive value5"
```

For example:

```yaml
sshd__match:
  - criteria: Group
    patterns: sftp-only
    options:
      - "AuthorizedKeysFile /etc/sftp-only/authorized_keys/%u"
      - "ChrootDirectory /srv/sftp-only/%u"
      - "ForceCommand internal-sftp -u 077"
      - "AllowTcpForwarding no"
  - criteria: User
    patterns:
      - root
      - admin
    options:
      - "PermitRootLogin yes"
      - "AuthenticationMethods publickey,password"
```

That will result in:

```
Match Group sftp-only
    AuthorizedKeysFile /etc/sftp-only/authorized_keys/%u
    ChrootDirectory /srv/sftp-only/%u
    ForceCommand internal-sftp -u 077
    AllowTcpForwarding no

Match User root,admin
    PermitRootLogin yes
    AuthenticationMethods publickey,password
```

Dependencies
------------

This role has no dependency.

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

Galaxy
------

To make use of this role as a galaxy role, put the following lines in
`requirements.yml`:

```yaml
- name: sshd_apply
  src: https://github.com/quidame/ansible-role-sshd_apply.git
  version: 0.2.0
  scm: git
```

and then

```bash
ansible-galaxy install -r requirements.yml
```

License
-------

GPLv3

Author Information
------------------

<quidame@poivron.org>
