# Monitor infrastructure

**Ansible scripts for deploying the service**

```sh
├── Vagrantfile        # Test the scripts locally with `vagrant up`
├── group_vars         # Common variables and deploy secrets
├── dev                # Inventory of hosts used in local dev
├── next               # Inventory of staging hosts
├── production         # Inventory of LIVE hosts
├── roles              # Define the tasks that set up a given role.
├── bootstrap.yml      # Playbook for getting new vms up to spec
└── deploy.yml         # Playbook for updating our app
```

[Ansible](http://docs.ansible.com/ansible/index.html) works by assigning roles to hosts.

- A **host** is any server in our infrastructure.
- A **role** can be things like `frontend`, `db`, etc.

Roles contain the tasks and and files to install and configure the services needed.

**e.g**: `frontend` _clones our app code, installs npm deps, and configures nginx as a proxy._

Key to making it work is ensuring tasks are idempotent. We can run all the tasks at any time. Either the task changes the system as required, or has no effect if that change is already in place.

An **inventory** defines named groups of servers. We use **playbooks** to assign roles those groups. We have a playbook that bootstraps a brand new vm to be used by ansible, which we assume will be run once on against each machine.

```sh
ansible-playbook -i production bootstrap.yml --extra-vars "ansible_user=root"
```

where
- `-i production` limits the hosts affected to just those listed in `production/inventory`
- `bootstrap.yml` is the playbook to run.
- `--extra-vars "ansible_user=root"` tells ansible to connect as `root` for this run. It's only needed while we don't have an ansible user.

[`bootstrap.yml`](https://github.com/tableflip/monitor-infrastructure/blob/master/bootstrap.yml) just steps up the ansible user and not much else.

```yaml
- hosts: app
  roles:
    - bootstrap
```

By assigning `app` hosts the role `bootstrap`, it's telling ansible to run the tasks defined in [`roles/boostrap/tasks/main.yml`](https://github.com/tableflip/monitor-infrastructure/blob/master/roles/bootstrap/tasks/main.yml)

```yaml
- name: Ensure base OS is up-to-date
  become: yes
  apt: upgrade=dist update_cache=yes

- name: Ensure ansible user exists
  become: yes
  user: name=ansible comment="Ansible" groups="ansible,sudo"
...
```

Once we have an `ansible` user, we can forget about about `bootstrap.yml`, and get on with setting up our roles, as defined in `playbook.yml`

At the start of a project, it's normal to have all the roles on the same host; a single vm dealing with the frontend, api and db, as it's then much easier to roll out additional VMs for staging and test.

When we need to scale the infrastructure we can add additional hosts to an inventory, to scale a roll horizontally across many identically configured servers, and we can split roles our to separate hosts, to create optimised VMs with a single purpose; e.g. a separate `db` server.

## Prerequisites

- Install VirtualBox
- Install Vagrant (`brew install vagrant`)
- Install Ansible
- You will need the vault password. Talk to your friendly neighborhood tableflipper.

## Usage

**To bootstrap a local test server with vagrant**

- Add `10.100.120.100	dev.monitor.tableflip.io` to your local `/etc/hosts`

```sh
# Download and provision a vm
vagrant up

# Update vm with our roles
ansible-playbook -i dev playbook.yml --ask-vault-pass
```

You now have a test vm, running locally

**To bootstrap a new production vm**

- Add the new remote to the relevant inventory
- Add your public ssh key in `/root/.ssh/authorized_keys` on the remote

```sh
# bootstrap ansible user
ansible-playbook -i production bootstrap.yml --ask-vault-pass --extra-vars "ansible_user=root"

# Intall app and dependencies
ansible-playbook -i production deploy.yml --ask-vault-pass
```

## Secrets - Ansible Vault

See: http://docs.ansible.com/ansible/playbooks_vault.html

**Creating Encrypting Files**

Encrypt a list of files. You'll be prompted for a passphrase which'll be the key for decrypting them.

```sh
ansible-vault encrypt [files]
```

For example, to encrypt our deploy keys and secrets, we do:

```sh
ansible-vault encrypt group_vars/all/secrets.yml group_vars/dev/dev_secrets.yml group_vars/next/next_secrets.yml group_vars/production/production_secrets.yml
```

**Editing Encrypted Files**

```sh
ansible-vault edit group_vars/production/production_secrets.yml
```

Will prompt you for the passphrase and open the file in your default $EDITOR as configured in your shell.
