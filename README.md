# SSH Remote Server Setup

Solution for the [SSH Remote Server Setup](https://roadmap.sh/projects/ssh-remote-server-setup) project from roadmap.sh.

**Goal:** set up a remote Linux server, create two SSH key pairs, and be able to log in with each of them, both with `ssh -i` and with an alias from `~/.ssh/config` (steps 1–4, 6).

Beyond the requirements:
- **Step 5 (extra):** password login and root login are disabled. Not required by the project, but a standard hardening practice.
- **Step 7 (optional):** fail2ban, the stretch goal of the project.

## Environment

| | |
|---|---|
| Client | macOS (built-in OpenSSH) |
| Server | VPS, Ubuntu 24.04 LTS |
| Server user | a regular user with `sudo` rights (not root) |

> **Note:** the instructions are written for macOS. The `UseKeychain` option in step 4 exists only in Apple's OpenSSH (see [Notes](#notes)).

Placeholders used below:
- `<user>` — the username on the server
- `<server_ip>` — the public IP address of the server

## How it works (short version)

SSH key authentication uses a **key pair**:
- the **private key** (`roadmap_key1`) stays on the Mac and is never shared;
- the **public key** (`roadmap_key1.pub`) is copied to the server into `~/.ssh/authorized_keys`.

When connecting, the server asks the client to sign some data with the private key and checks the signature with the public key. The private key never leaves the client machine.

Two separate key pairs are used to mirror real-world practice: one key per device or purpose, so a single key can be revoked without affecting the others.

---

## Step 1. Create two key pairs (on the Mac)

Check existing keys first so nothing gets overwritten:

```bash
ls ~/.ssh/
```

Generate two ed25519 key pairs:

```bash
ssh-keygen -t ed25519 -C "key1-comment" -f ~/.ssh/roadmap_key1
ssh-keygen -t ed25519 -C "key2-comment" -f ~/.ssh/roadmap_key2
```

- `-t ed25519` — modern key type: short, fast and secure
- `-C` — a comment that helps identify the key later in `authorized_keys`
- `-f` — the file name (without it the key would go to `~/.ssh/id_ed25519`)

Each key is protected with a **passphrase**, so a stolen key file is useless on its own.

Result: `roadmap_key1`, `roadmap_key1.pub`, `roadmap_key2`, `roadmap_key2.pub`.

## Step 2. Copy the public keys to the server

```bash
ssh-copy-id -i ~/.ssh/roadmap_key1.pub <user>@<server_ip>
ssh-copy-id -i ~/.ssh/roadmap_key2.pub <user>@<server_ip>
```

`ssh-copy-id` asks for the user's password one last time, appends the key to `~/.ssh/authorized_keys` on the server, and sets the correct permissions (`700` for `~/.ssh`, `600` for `authorized_keys`).

> Only the **`.pub`** files are copied. Private keys never leave the Mac.

## Step 3. Log in with each key separately

```bash
ssh -i ~/.ssh/roadmap_key1 -o IdentitiesOnly=yes <user>@<server_ip>
ssh -i ~/.ssh/roadmap_key2 -o IdentitiesOnly=yes <user>@<server_ip>
```

`IdentitiesOnly=yes` makes SSH use **only** the key given with `-i`. Without it, ssh-agent may offer every loaded key, and it would be unclear which one actually worked.

To see which key the server accepted, add `-v` and look for `Server accepts key: ...`.

## Step 4. Create an SSH config file (on the Mac)

```bash
nano ~/.ssh/config
chmod 600 ~/.ssh/config
```

```
Host srv1
    HostName <server_ip>
    User <user>
    IdentityFile ~/.ssh/roadmap_key1
    IdentitiesOnly yes

Host srv2
    HostName <server_ip>
    User <user>
    IdentityFile ~/.ssh/roadmap_key2
    IdentitiesOnly yes

Host *
    AddKeysToAgent yes
    UseKeychain yes
```

- `srv1` / `srv2` — short aliases, one per key
- `AddKeysToAgent yes` + `UseKeychain yes` — the passphrase is stored in the macOS Keychain, so it does not have to be typed on every login

Now connecting is just:

```bash
ssh srv1
ssh srv2
```

## Step 5 (extra). Disable password and root login (on the server)

> This step is not required by the project. It is a common hardening practice for any server with a public IP.

> Do this **only after** both keys are confirmed to work, and keep one SSH session open until the result is verified. Otherwise you can lock yourself out of the server.

Instead of editing the main `/etc/ssh/sshd_config`, a separate drop-in file is created:

```bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

```
PasswordAuthentication no
PermitRootLogin no
```

Why a drop-in file with the `00-` prefix:
- files in `sshd_config.d/` are read in alphabetical order, and **sshd uses the first value it finds** for each option;
- on Ubuntu, `50-cloud-init.conf` may already contain `PasswordAuthentication yes`. `00-hardening.conf` is read earlier, so it wins;
- custom changes are kept separate from the distribution's config and survive package updates.

Check the syntax and the effective values, then apply:

```bash
sudo sshd -t                                          # no output = config is valid
sudo sshd -T | grep -iE 'passwordauth|permitroot'     # expected: passwordauthentication no, permitrootlogin no
sudo systemctl restart ssh                            # on Ubuntu the service is called "ssh", not "sshd"
```

> `sshd -T` reads the config files on disk, not the running process, so the restart is still needed to apply the changes.

## Step 6. Verify (from the Mac, in a new terminal)

```bash
ssh srv1                                            # logged in with key 1
ssh srv2                                            # logged in with key 2

ssh -o PubkeyAuthentication=no <user>@<server_ip>   # Permission denied (publickey).
ssh root@<server_ip>                                # Permission denied (publickey).
```

`Permission denied (publickey)` means the server now only accepts key-based authentication.

## Step 7 (optional). Protect SSH with fail2ban (on the server)

> Optional: this is the stretch goal of the project. The main task is complete after step 6.

Any server with a public IP is constantly scanned by bots that try common usernames and passwords on port 22. You can see it in the log:

```bash
sudo journalctl -u ssh --since today | grep -i "invalid user" | head -20
```

Password login is already disabled (step 5), so these attempts cannot succeed. fail2ban adds one more layer (defense in depth): it reduces log noise and load, and the same tool can later protect other services (nginx, web apps with a login form).

**How it works:** fail2ban reads the service logs, matches failed login lines with regex **filters**, counts failures per IP, and when an IP fails `maxretry` times within `findtime`, it **bans** that IP with a firewall rule for `bantime`. The ban is lifted automatically. One "filter + action + settings" set for a service is called a **jail**.

Install:

```bash
sudo apt update
sudo apt install fail2ban
```

Do **not** edit `/etc/fail2ban/jail.conf`: it is overwritten by package updates. Custom settings go into `jail.local` (same idea as the sshd drop-in file in step 5):

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled  = true
backend  = systemd
maxretry = 5
findtime = 10m
bantime  = 1h
```

- `backend = systemd` — read logs from journald (the default logging system on Ubuntu 24.04)
- `maxretry = 5`, `findtime = 10m` — 5 failed attempts within 10 minutes from one IP trigger a ban
- `bantime = 1h` — the IP is banned for 1 hour

`ignoreip` is not set: the server's own addresses are already excluded by `ignoreself = true` (enabled by default). A dynamic home IP should not be whitelisted either: the address may later belong to someone else.

Start the service, enable it on boot and check the jail:

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd        # failed attempts, banned IPs
```

If you ban yourself, log in through the provider's web console and unban your IP:

```bash
sudo fail2ban-client set sshd unbanip <your_ip>
```

---

## Troubleshooting / lessons learned

**Edited the wrong file.** At first I opened `/etc/sshd_config` instead of `/etc/ssh/sshd_config`. `nano` silently created a new empty file that sshd never reads, so nothing changed. `sudo sshd -T` showed the real effective values and caught the mistake.
→ Use Tab completion for system paths: if a path does not autocomplete, it is wrong. Always check the effective config with `sshd -T`.

**Settings in `sshd_config` are ignored.** Check for overrides in `/etc/ssh/sshd_config.d/` (e.g. `50-cloud-init.conf`) and remember that the first value wins. Lines starting with `#` are comments and have no effect.

**The key is not accepted and the password is requested instead.** Usually caused by wrong permissions. sshd ignores `authorized_keys` if it is writable by others. Required: `~/.ssh` → `700`, `~/.ssh/authorized_keys` → `600`. The server log shows the reason: `sudo journalctl -u ssh`.

**The server stopped responding to SSH after several failed logins.** Your IP was probably banned by fail2ban. Use the provider's web console, check `sudo fail2ban-client status sshd` and unban with `sudo fail2ban-client set sshd unbanip <your_ip>`.

**`REMOTE HOST IDENTIFICATION HAS CHANGED`.** The server's host key does not match `~/.ssh/known_hosts`. This is expected after reinstalling the server OS (`ssh-keygen -R <server_ip>` removes the old entry). Otherwise, find out why before connecting.

## Notes

- `UseKeychain` is macOS-only. On Linux, add `IgnoreUnknown UseKeychain` at the top of `~/.ssh/config`, or remove the line.
- Private keys, real IP addresses and usernames are not committed to this repository.
