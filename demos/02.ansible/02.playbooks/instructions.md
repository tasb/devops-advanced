# Demo: Run playbook

## Run playbook on dev env

```bash
ansible-playbook -i inventory/dev webserver.yml
```

## Run playbook on prod env

```bash
ansible-playbook -i inventory/prod webserver.yml
```

## Run cleanup playbook on all envs

```bash
ansible-playbook -i inventory webserver-clean.yml
```
