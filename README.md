# Ansible Project — Full Setup Walkthrough

Two machines involved:
- **[WSL]** = the WSL Ubuntu terminal on your Windows PC. This is the *control
  node* — where you install Ansible and run `ansible-playbook`.
- **[EC2]** = your EC2 instance. Ansible connects to it over SSH; you never
  run Ansible commands here directly.

Do every step in order. Don't skip ahead.

---

## Step 0 — Open the right terminal

Open **WSL (Ubuntu)**, not PowerShell, not Git Bash. If you don't have it yet:

**[Windows PowerShell, as Administrator]**
```powershell
wsl --install -d Ubuntu
```
Restart if prompted, then open the "Ubuntu" app from the Start menu. All
steps from here on are **[WSL]** unless stated otherwise.

---

## Step 1 — Install Ansible

**[WSL]**
```bash
sudo apt update
sudo apt install -y ansible git
ansible --version
```

---

## Step 2 — Move the project into WSL's own filesystem

**[WSL]**
```bash
mkdir -p ~/projects
cp -r "/mnt/d/Learning/Configuration Management" ~/projects/ansible-config-mgt
cd ~/projects/ansible-config-mgt
```

From now on, **every command in this guide runs from inside this folder**,
unless marked otherwise.

---

## Step 3 — Install the Ansible collections the roles use

**[WSL]**
```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Step 4 — Copy your EC2 `.pem` key into WSL

Find your `.pem` file (the one you downloaded from AWS when you created the
EC2 instance). Copy it into WSL's own filesystem. 

**[WSL]**
```bash
mkdir -p ~/.ssh
cp /mnt/d/path/to/your-key.pem ~/.ssh/ec2-key.pem # you can use sudo if you get permission denied message.
chmod 600 ~/.ssh/ec2-key.pem
```

Test the raw SSH connection before touching Ansible at all — this
confirms the key works and also caches the host's fingerprint so Ansible
won't fail on host-key verification later:

**[WSL]**
```bash
ssh -i ~/.ssh/ec2-key.pem ec2-user@<your-ec2-public-ip>
```
(Use `ec2-user` if your instance is Amazon Linux. Use `ubuntu` instead if
it's an Ubuntu AMI.)

Type `yes` when asked about the host fingerprint. If you land at a shell
prompt on the EC2 box, you're good — type `exit` to come back.

**If this step fails, stop here and fix it before continuing.** Nothing
past this point will work if plain SSH doesn't.

---

## Step 5 — Generate a keypair for the `ssh` role

This is a *separate* key from your `.pem` — it's the one the `ssh` role
will provision on the server, demonstrating the role actually granting
access rather than just replaying AWS's own key.

**[WSL]**
```bash
ssh-keygen -t ed25519 -C "user-ansible" -f ~/.ssh/ansible_key -N ""
cat ~/.ssh/ansible_key.pub
```

Copy the full line it prints (starts with `ssh-ed25519 AAAA...`). You'll
paste it into `group_vars/all.yml` in Step 7.

---

## Step 6 — Get a static site tarball

The `app` role needs a `.tar.gz` of some static HTML site. Pick **one**:

**Option A — use a placeholder site (fastest, fine for this project)**
```bash
mkdir -p ~/sample-site
cat > ~/sample-site/index.html << 'EOF'
<!DOCTYPE html>
<html><head><title>Deployed via Ansible</title></head>
<body><h1>Deployed via Ansible</h1><p>It works.</p></body></html>
EOF
tar -czf roles/app/files/website.tar.gz -C ~/sample-site .
```

**Option B — use a real site folder you already have**
Replace `~/path/to/your/site` with wherever that site's `index.html`
actually lives:
```bash
tar -czf roles/app/files/website.tar.gz -C ~/path/to/your/site .
```

Either way, confirm the tarball was created and contains files:
```bash
tar -tzf roles/app/files/website.tar.gz
```
You should see `./` and `./index.html` (and any other site files) listed.

---

## Step 7 — Edit your two config files

**[WSL, using nano/vim, or edit from Windows and it'll still be visible in WSL]**

### 7a. `inventory.ini`
```ini
[webservers]
web1 ansible_host=<your-ec2-public-ip> ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/ec2-key.pem

[webservers:vars]
ansible_python_interpreter=/usr/bin/python3
```
Use `ubuntu` instead of `ec2-user` if that's your AMI's default user.

### 7b. `group_vars/all.yml`
Update just these two values (leave the rest as-is):
```yaml
authorized_public_keys:
  - "<your-public-key>"

app_tarball_filename: "website.tar.gz"
```

---

## Step 8 — Run it

**[WSL]**, from inside `~/projects/ansible-config-mgt`:

```bash
# Dry run first, to catch obvious mistakes without changing anything
ansible-playbook -i inventory.ini setup.yml --check

# Real run, everything
ansible-playbook -i inventory.ini setup.yml

# Or just one role at a time, while debugging
ansible-playbook -i inventory.ini setup.yml --tags "base"
ansible-playbook -i inventory.ini setup.yml --tags "ssh"
ansible-playbook -i inventory.ini setup.yml --tags "nginx"
ansible-playbook -i inventory.ini setup.yml --tags "app"
```

---

## Step 9 — Verify

Open in a browser:
```
http://<your-ec2-public-ip>/
```
You should see your site. If it times out (not "connection refused"), it's
almost always the **EC2 Security Group** missing an inbound rule for port
80 — check that in the AWS console, not in Ansible.


[https://github.com/46h15h3k/Ansible-Configuration-Management.git](https://roadmap.sh/projects/configuration-management)
