# Demo: Run Ad-hoc commands

## Check smaller Ansible config file

Open the file ansible.cfg in the current directory

## Run ping on all remote host

```bash
ansible all -m ansible.builtin.ping
```

## Run copy file on nodes pattern

```bash
ansible nodes -m ansible.builtin.copy -a "src=/etc/hosts dest=/tmp/hosts"
```

## Check if file was copied

```bash
ansible nodes -m ansible.builtin.command -a "cat /tmp/hosts"
```

## Install nginx-light

```bash
ansible webserver -m ansible.builtin.apt -a "name=nginx-light state=present" --become
```

## Check if nginx service is running

```bash
ansible webserver -m ansible.builtin.command -a "systemctl status nginx"
```

## Stop nginx service with error

```bash
ansible webserver -m ansible.builtin.service -a "name=nginx state=stopped"
```

## Stop nginx service

```bash
ansible webserver -m ansible.builtin.service -a "name=nginx state=stopped" --become
```

## Check if nginx service is running after stop

```bash
ansible webserver -m ansible.builtin.command -a "systemctl status nginx"
```

## Uninstall nginx-light

```bash
ansible webserver -m ansible.builtin.apt -a "name=nginx-light state=absent" --become
```
