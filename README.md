# Ansible server setup

Ansible playbooks for setting up and maintaining Linux servers. They install
common tools, create a user account, configure SSH and firewall rules, and apply
a few settings for small virtual servers.

The project includes tasks for Debian, Red Hat, and Alpine systems.
It is mostly targeting debian, and the others are not that tested.

## The two main playbooks

[update.yaml](playbooks/update.yaml)

Updates theos packages using apt/apk/yum depending on the OS. Makes it easy for me to update all server all at once regularly.

[setup_server.yaml](playbooks/setup_server.yaml)

I use it to initialize a new server after installation. It runs these sections:

| Section | What it does |
| --- | --- |
| Install packages | Installs common command-line tools and packages for the server's operating system. Can also copy downloaded programs if provided. I keep a directory of github binaries like ripgrep, fd, lsd and copy it over to the new server. |
| Configure system logs | Limits log storage on servers using systemd. |
| Configure the firewall | Adds SSH, VPN, and trusted-network rules for UFW on Debian systems. Also blocks outgoing mail on port 25. |
| Update firewall rules | Adds the configured trusted IP addresses and VPN networks to UFW. |
| Configure user accounts | Creates your chosen account, gives it sudo access, and generates an SSH key. Optional tasks add shell shortcuts and SSH-key authentication for sudo. |
| Configure SSH | Adds your public keys and sets which accounts can log in. Applies the SSH settings in the template. |
| Configure restic access | Allows your chosen account to run the restic backup program with sudo without a password. Does not install restic or create backup jobs. |
| Enable BBR | Attempts to enable BBR, a Linux setting that can improve network performance. Skips OpenVZ and LXC guests. |
| Disable unwanted services | Stops a selected list of background services on Debian systems. |

Normal setup adds UFW rules but does not enable UFW or run the optional cleanup,
shell, or sudo authentication tasks. See the optional commands below.

## Before you start

Run the commands from the project's main directory on a machine with Ansible
installed. The servers need SSH access and Python 3 already available.

Install the collections used by these playbooks:

```bash
ansible-galaxy collection install ansible.posix community.general
```

The playbooks connect as root by default. For a server that uses another login
account, set `ansible_user` to that account and `ansible_become=True` in `hosts`.
That account needs permission to use sudo. Add `--ask-become-pass` to commands if
sudo requires a password.

## Adapt the project to your servers

### Server list and account settings

Copy the example inventory, then edit `hosts`:

```bash
cp hosts.example hosts
```

Replace the example server names and IP addresses with your own. Keep the servers
you want to manage in the `vps` group, either directly or through its child groups.
You can set shared values for a group or different values for individual servers.

| Setting in `hosts` | What to change |
| --- | --- |
| `my_user` | Set the account you want to create and manage. Required for setup and account configuration. |
| `ansible_user` | Set the existing account Ansible uses to connect. This can differ from `my_user`. |
| `ansible_port` | Set the SSH port you connect to if it is not 22. The example also uses `server.example.com:2123` to set this port. |
| `ssh_listen_port` | Can be used to open an alternate SSH port in UFW. |
| `ssh_port_22` | Set `True` where port 22 should also be allowed. This matters for servers whose provider forwards an external port to port 22. |
| `low_mem` | Set `True` to use smaller system logs on servers with limited resources. |
| `my_user_uid`, `my_user_gid` | Optional numeric account and group IDs. Leave them out to use the system's normal choices. |
| `bak_user` | Optional extra account to allow through SSH. The account must already exist. |

For example, if you connect on port 2123 but your provider forwards it to port 22
on the server, use:

```ini
server.example.com:2123 ansible_user=root ssh_listen_port=22 ssh_port_22=True
```

These settings add firewall allowances; they do not change the SSH listening port.
Before enabling UFW, make sure its rules allow the port your server actually uses.

### Your SSH keys

Replace all the public-key URLs in the example inventory with URLs containing
your own public keys. Remove or comment out any key setting you do not need.

| Setting | Purpose |
| --- | --- |
| `ssh_allowed_public_keys_my_user` | Public keys allowed to log in as `my_user`. |
| `ssh_allowed_public_keys_root` | Public keys allowed to log in as root. |
| `ssh_public_key_ansible` | Public key for the machine running Ansible. |
| `ssh_public_key_url` | Public key for the optional SSH-key authentication using pam for sudo commands. |

If a URL setting is missing, its key-installation task is skipped. Other SSH
settings still apply.

The default connection key is `~/.ssh/ansible`. Change the key path in
`ansible.cfg` and any playbook that sets `ansible_ssh_private_key_file` if you use
a different key. In `ansible.cfg`, set `host_key_checking=True` if you want
Ansible to verify server identities when connecting.

### Account passwords

Create an encrypted file for the account password and generated SSH key's
passphrase:

```bash
ansible-vault create vars/vault
```

Enter these values in the editor:

```yaml
vault_user_passwd: "<Linux password hash>"
vault_ssh_key_passphrase: "<passphrase for the generated SSH key>"
```

Use a Linux password hash for `vault_user_passwd`, rather than a plain password.
Both values are required for account creation. Use `ansible-vault edit vars/vault`
to change them later.

### Packages, networks, and other preferences

Review these files to match your needs:

| Files | What to review |
| --- | --- |
| `playbooks/setup/install-programs.yml` | Add or remove packages for each operating system. |
| `playbooks/setup/firewall-ufw.yml`, `playbooks/setup/ufw_rules_update.yml`, `playbooks/reconfigure/ufw_rules_update.yml` | Trusted IP addresses and VPN networks. Outgoing port 25 blocked. |
| `playbooks/individual/firewall.yaml` | Review the Red Hat firewall rules, including the fixed SSH port 2222 and trusted networks. This playbook configures an existing firewall service. I mostly use debian systems and so haven't tested it on firewalld for many changes. |
| `playbooks/setup/ssh-config.yml`, `templates/sshd_config` | Review allowed accounts, including `tun-user`, and the networks where root and password login are allowed. Keep these networks consistent with the firewall rules. |
| `playbooks/setup/journald.yml` | Configuring journald storage space for system logs. |
| `templates/sudo_restic` | Review password-free backup access and the path to restic. I use restic wth sudo to take backups. Remove the restic section from setup if you do not need it. |
| `playbooks/setup/disable_services.yml`, `playbooks/reconfigure/disable_services.yml` | Remove services from the stop list if your server needs them. |
| `templates/bashrc`, `templates/bash_aliases`, `templates/bash_aliases_systemd` | Use your own bash shell aliases. |
| `playbooks/update.yaml` | By default, all debian servers reboot when needed, except those in the `hdd` group. I use luks disk encrypted mounts and so reboot manually. |

To skip a main setup section entirely, remove or comment out its task in
`playbooks/setup_server.yaml`. You can also use the individual playbooks below
when you only need one part of the setup.

## Run the playbooks

Start with one server or group. For example, set up the `cloud` group:

```bash
ansible-playbook -i hosts playbooks/setup_server.yaml --ask-vault-pass --limit cloud
```

Remove `--limit cloud` to set up all servers in `vps`. You can also use a server
name instead of a group with `--limit`.

Other commands:

```bash
# Update system packages; may reboot Debian servers when needed
ansible-playbook -i hosts playbooks/update.yaml

# Remove unused Debian packages
ansible-playbook -i hosts playbooks/update.yaml --tags clean

# Configure firewall rules
ansible-playbook -i hosts playbooks/individual/firewall.yaml

# Configure SSH
ansible-playbook -i hosts playbooks/individual/ssh-config.yaml

# Create or configure the user account
ansible-playbook -i hosts playbooks/individual/deploy-user-config.yaml --ask-vault-pass

# Try enabling BBR
ansible-playbook -i hosts playbooks/individual/enable_bbr.yaml

# Update UFW trusted networks and disable selected services
ansible-playbook -i hosts playbooks/individual/reconfigure.yaml
```

To copy your downloaded programs into `/usr/local/bin/` during setup, add
`-e 'github_download_dir=/path/to/programs/'`. Use a folder on the machine running
Ansible, with a trailing `/`. Leave this setting out to skip copying programs.

## Optional actions

These actions run only when you choose their tags. Review the relevant settings
first. For firewall changes, keep a way to reach the server through your
provider's console and confirm SSH access after the change.

```bash
# Enable UFW firewall
ansible-playbook -i hosts playbooks/individual/firewall.yaml --tags ufw_enable

# Delete the old UFW rules listed in the task file
ansible-playbook -i hosts playbooks/individual/firewall.yaml --tags ufw_delete

# Apply shell shortcuts for root and your configured user
ansible-playbook -i hosts playbooks/individual/deploy-user-config.yaml --ask-vault-pass --tags bash

# Allow SSH-key authentication for sudo commands instead of asking password
ansible-playbook -i hosts playbooks/individual/deploy-user-config.yaml --ask-vault-pass --tags sudo_pam
```

Keep `ssh_port_22=True` if you still need port 22; the UFW cleanup tag can remove
its rule otherwise.

