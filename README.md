# Ansible Role: Rsyslog

[![Build Status](https://travis-ci.com/hadret/ansible-role-rsyslog.svg?branch=master)](https://travis-ci.com/hadret/ansible-role-rsyslog)

Installs and configures [rsyslog](https://rsyslog.com) on Debian/Ubuntu servers.

This role installs and configures the latest version of rsyslog from official APT repository (on Debian) or [official PPA](https://launchpad.net/~adiscon/+archive/ubuntu/v8-stable) (on Ubuntu). By default it will take over contents of the `/etc/rsyslog.conf` and the `/etc/rsyslog.d/50-default.conf`.

## Requirements

None.

## Role variables

Here are available variables with their default values (as in
[defaults/main.yml](defaults/main.yml)):

```yaml
rsyslog_rules: []
```

An array of rules for rsyslog. Each entry will create a separate config file named as `$priority-$rule_name.conf`. Be sure to check `defaults/main.yml` for commented out example with all of the available options.

```yaml
rsyslog_rules:
  - rule_name: "remote-udp"
    priority: 99
    ruleset: |
      module(load="omfwd")
      action(type="omfwd" target="central.server.local" port="514" protocol="udp")
    state: "present"
```

Here's an example of a fully-populated `rsyslog_rules` entry. Please note the `|` for the block declaration for the `ruleset`. From there it becomes _bare_ rsyslog config syntax.

Alternative route from defining the `rsyslog_rules` in a rule-by-rule manner would be to use the `rsyslog_extra_conf_options`. It then extends the main `/etc/rsyslog.conf` configuration file with extra options instead of creating new files in the `/etc/rsyslog.d`.

```yaml
rsyslog_extra_conf_options: |
  module(load="imudp")
  input(type="imudp" port="514")
```

Here too `|` is used for the block declaration and the code itself is a _bare_ rsyslog config syntax. It can also be combined with `rsyslog_remove_default_rules: true` that would ensure `/etc/rsyslog.d/` is empty.

Additionally there are currently three preconfigured rsyslog rules. All of them have special, dedicated templates (`templates/*.conf.j2`). Only one of them is enabled by default and is called, well, `default`. It takes over definition of `/etc/rsyslog.d/50-default.conf` file. It can be easily disabled by specifying `state: "absent"`.

```yaml
rsyslog_rule_default:
  rule_name: "default"
  priority: 50
  template: "default.conf.j2"
```

Second one is `docker` that handles logs for the Docker containers running on the given host. It's being defined in the `/etc/rsyslog.d/20-docker.conf` file.

```yaml
rsyslog_rule_docker:
  rule_name: "docker"
  priority: 20
  template: "docker.conf.j2"
rsyslog_rule_docker_tag_all: true
```

It will create `/var/log/docker` and put log files inside it with `$CONTAINER_NAME.log` naming scheme. It expects that the `$syslogtag` will have `docker/` in the name (check the example below), otherwise it will push all the logs into `/var/log/docker/no_tag.log`. Additionally there's a `rsyslog_rule_docker_tag_all` that can be turned on when there's more than one container running on the given host and allows for single file with logs aggregated from all of them in `/var/log/docker/all.log` (*Note: this will double the space needed for the container logs*). You may check my [hadret.containers](https://github.com/hadret/ansible-role-containers) role for an example of the container definition with syslog support enabled.

```yaml
containers:
  - name: cadvisor
    image: "google/cadvisor:latest"
    state: started
    log_driver: journald
    log_options:
      tag: docker/cadvisor
```

`journald` is these days picked up automatically by rsyslog.

Last but not least is the `remote` handling. I wanted to create a turn-key solution for handling both client and server parts. Remote logging is currently very raw and basic, but it does work out of the box with a minimal configuration.

```yaml
rsyslog_rule_remote:
  rule_name: "remote"
  role: server
  priority: 99
  template: "remote.conf.j2"
  ruleset_name: "remote"
```

At least one remote protocol (`relp`/`tcp`/`udp`) has to be specified (*Note:  there's no default and specifying `rsyslog_rule_remote` alone will fail*). The way server-side is handled requires defined `ruleset_name` as it is `ruleset` that is executing the actual action of writing down the logs (via `omfile`) and applying predefined templates. These are configured to be as similar to the "ordinary" rules as possible with following outputs predefined: `auth.log`, `syslog.log`, `rsyslog.log`, `kern.log` and `mail.log`.

```yaml
rsyslog_rule_remote_relp:
  port: 514
```

Currently only `relp` supports TLS setup.

```yaml
rsyslog_rule_remote_relp:
  address: 0.0.0.0
  port: 514
  tls: true
  tls_cacert: "/tls-certs/ca.pem"
  tls_mycert: "/tls-certs/cert.pem"
  tls_myprivkey: "/tls-certs/key.pem"
  tls_authmode: "fingerprint"
```

Both `tcp` and `udp` currently allow only for specifying `address` (optional in server mode), `target` (required in client mode) and `port` (required in both modes).

```yaml
rsyslog_rule_remote_tcp:
  address: 0.0.0.0
  port: 514

rsyslog_rule_remote_udp:
  address: 0.0.0.0
  port: 514
```

Please note that you **can** define all three of them, with different addresses and ports (but each of them only once). All of the configuration will by default land in `/etc/rsyslog.d/99-remote.conf` (both server and client). It is currently impossible to have a single machine acting as both server and client solely with `rsyslog_rule_remote_relp` usage, but it is doable to specify additional rule with either `rsyslog_extra_conf_options` or `rsyslog_rules`.

```yaml
rsyslog_rule_remote:
  rule_name: "server"
  role: server
  priority: 99
  template: "remote.conf.j2"
  ruleset_name: "server"

rsyslog_rule_remote_udp:
  port: 514

rsyslog_rules:
  - rule_name: "client"
    priority: 99
    ruleset: |
      module(load="omfwd")
      action(type="omfwd" target="central.server.local" port="514" protocol="tcp")
```

*Note: all three of these additionally preconfigured rsyslog rules are dictionaries, not arrays. Only the `rsyslog_rules` allow for multiple rule definitions.*

## Extending and replacing templates

I realize that not everything is covered with variables and there are tons of different possible configuration options out there. That's the reason why I'm using templates for all of the rules which allows for easy extending, block replacing (via [Jinja2 template inheritance](http://jinja.pocoo.org/docs/2.9/templates/#template-inheritance)) or full template exchanging to match the needs I haven't thought of.

```yaml
rsyslog_conf_template: "rsyslog.conf.j2"
rsyslog_rules_template: "rules.conf.j2"
```

It can also be changed on per-rule basis.

```yaml
rsyslog_rule_default:
  rule_name: "default"
  priority: 50
  template: "{{ playbook_dir }}/templates/custom-default.conf.j2"

rsyslog_rule_docker:
  rule_name: "docker"
  priority: 20
  template: "{{ playbook_dir }}/templates/custom-docker.conf.j2"

rsyslog_rules:
  - rule_name: "remote-udp"
    priority: 90
    template: "{{ playbook_dir }}/templates/custom-udp.conf.j2"
  - rule_name: "remote-tcp"
    priority: 91
    template: "{{ playbook_dir }}/templates/custom-tcp.conf.j2"
```

### Example: extending modules block in main config file

`rsyslog_conf_template` has to be set to point to the new file in your playbook directory.

```yaml
rsyslog_conf_template: "{{ playbook_dir }}/templates/custom-rsyslog.conf.j2"
```

Custom template file has to be placed relative to your `playbook.yml`.

```
{% extends 'roles/external/hadret.rsyslog/templates/rsyslog.conf.j2' %}

{% block modules %}
$ModLoad imuxsock
$ModLoad imklog
$ModLoad immark

$ModLoad imudp
$UDPServerRun 514

$ModLoad imtcp
$InputTCPServerRun 514
{% endblock %}
```

The above example replaces/extends `modules` block in the main rsyslog config file.

## Dependencies

None.

## Example playbook

```
hosts: all
  roles:
    - hadret.rsyslog
```

## License

MIT.

## Authors

This role was somewhat assembled in 2019 by [Filip Chabik](https://chabik.com).
