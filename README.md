# Ansible Server Configuration Management

An Ansible playbook that fully configures a Linux server — base hardening,
SSH key provisioning, nginx installation, and static site deployment — in
one repeatable, idempotent run. Built as part of the
[roadmap.sh DevOps](https://roadmap.sh/projects/configuration-management) curriculum.

## What it does

Running `setup.yml` against a fresh server will:

1. **`base`** — update all packages, install common utilities (curl, git,
   htop, unzip, etc.), install and enable `fail2ban` with a jail config,
   and set the system timezone.
2. **`ssh`** — add the given public key(s) to the target user's
   `authorized_keys`, and optionally harden `sshd_config` (disable root
   login / password auth).
3. **`nginx`** — install nginx, deploy a server block pointing at the
   site's deploy directory, open the firewall for HTTP, and start the
   service.
4. **`app`** — upload a tarball of a static site and unpack it into the
   nginx web root. A stretch-goal mode clones the site straight from a
   GitHub repo instead.

Each role is idempotent — re-running the playbook makes no changes on a
server that's already configured correctly.

## Repo structure

```
.
├── ansible.cfg
├── inventory.ini
├── requirements.yml
├── setup.yml
├── group_vars/
│   └── all.yml
└── roles/
    ├── base/
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   ├── defaults/main.yml
    │   └── templates/jail.local.j2
    ├── ssh/
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── defaults/main.yml
    ├── nginx/
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/nginx.conf.j2
    └── app/
        ├── tasks/main.yml
        ├── defaults/main.yml
        └── files/          # put your website.tar.gz here
```

Roles work on both Debian/Ubuntu (`apt`) and RHEL/Amazon Linux (`dnf`)
targets — useful since I run Amazon Linux 2023 for other projects in this
series but the roadmap.sh brief assumes a generic Linux box.

## Prerequisites

- Ansible installed locally (`pip install ansible` or your package
  manager's build).
- A running Linux server (EC2, DigitalOcean droplet, etc.) reachable over
  SSH with a key you already hold.
- Collections used by a couple of tasks (`ufw`, `firewalld`, `timezone`):

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

## Setup

1. Edit **`inventory.ini`** with your server's IP/hostname, SSH user, and
   key path:

   ```ini
   [webservers]
   web1 ansible_host=203.0.113.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
   ```

2. Edit **`group_vars/all.yml`**:
   - `authorized_public_keys` — your public key(s) for the `ssh` role.
   - `app_tarball_src` — path to your static site tarball (default
     `roles/app/files/website.tar.gz`), **or** set
     `app_deploy_method: git` and fill in `app_git_repo` / `app_git_version`
     for the stretch-goal flow.

3. Drop your site tarball in `roles/app/files/` (unless using the git
   method):

   ```bash
   tar -czf roles/app/files/website.tar.gz -C /path/to/your/site .
   ```

## Usage

```bash
# Run every role
ansible-playbook setup.yml

# Run a single role
ansible-playbook setup.yml --tags "app"
ansible-playbook setup.yml --tags "nginx"

# Run a subset
ansible-playbook setup.yml --tags "ssh,nginx"

# Dry run first (recommended)
ansible-playbook setup.yml --check --diff
```

After a successful run, the site is live at `http://<server-ip>/`.

## Deploy from GitHub

Set in `group_vars/all.yml`:

```yaml
app_deploy_method: "git"
app_git_repo: "https://github.com/your-username/your-static-site.git"
app_git_version: "main"
```

Then `ansible-playbook setup.yml --tags "app"` clones (or fast-forwards)
the repo directly into the nginx web root instead of unpacking a tarball
— handy for pushing updates without re-uploading a new archive each time.

## Notes / gotchas

- If `PasswordAuthentication no` locks you out because your key isn't
  actually working yet, leave `ssh_disable_password_auth: false` in
  `roles/ssh/defaults/main.yml` until you've confirmed key-based login,
  then flip it on and re-run with `--tags ssh`.
- On EC2, remember the instance's **Security Group** also needs an inbound
  rule for port 80 — `ufw`/`firewalld` rules inside the box won't help if
  the cloud-level firewall is blocking it first.
- `--check --diff` won't fully dry-run tasks that use `unarchive` or
  `git` (they report "would have changed" without inspecting archive
  contents), so treat that mode as a sanity check, not a guarantee.
