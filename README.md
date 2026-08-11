# Linux Server Setup & Hardening

A practical project for taking a fresh Ubuntu Linux server and configuring a secure baseline before deploying applications.

## Project Objective

The goal is to configure and harden a fresh Ubuntu server by:

- Creating a non-root administrative user with sudo privileges
- Configuring SSH key-based authentication
- Disabling SSH password authentication
- Configuring UFW firewall
- Allowing only required network ports
- Updating system packages
- Configuring automatic security updates
- Installing and configuring Fail2Ban
- Setting the correct timezone
- Setting a meaningful hostname
- Managing services with systemctl
- Inspecting logs with journalctl and /var/log
- Performing a final security verification

## Overall Security Flow

```text
                         INTERNET
                            |
                            v
                    +---------------+
                    |      UFW      |
                    |   Firewall    |
                    +-------+-------+
                            |
                       SSH : 22
                            |
                            v
                    +---------------+
                    |      SSH      |
                    |  Key-Based    |
                    | Authentication|
                    +-------+-------+
                            |
                            v
                    +---------------+
                    | Non-Root User |
                    |   + sudo      |
                    +-------+-------+
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
          systemctl     journalctl     /var/log
          Services        Logs           Files

Additional Security:
- Fail2Ban
- Automatic Security Updates
- Meaningful Hostname
- Correct Timezone
```

---

# 1. Initial Server Access

The project assumes a fresh Ubuntu server has been provisioned by a cloud provider such as AWS, DigitalOcean, Linode, or another VPS provider.

Initial login may be performed using the provider's initial credentials or SSH configuration:

```bash
ssh root@SERVER_IP
```

Replace `SERVER_IP` with the server's actual IP address.

Root should be used only for initial setup. Day-to-day administration should be performed using a non-root user with sudo privileges.

---

# 2. Why Avoid Direct Root Usage?

The root user has unrestricted privileges.

If an administrator accidentally runs a destructive command as root, the system can be seriously damaged.

Instead, use:

```text
Normal User
     |
     +---- sudo ----> Privileged Command
```

For example:

```bash
sudo systemctl restart nginx
```

This makes privileged operations explicit.

## Interview Answer

> Root has unrestricted privileges, so I use a normal administrative user with sudo for day-to-day administration. This follows the principle of least privilege and reduces the risk of accidental system-wide damage.

---

# 3. Update the System

First refresh the package information:

```bash
sudo apt update
```

Then install available package updates:

```bash
sudo apt upgrade -y
```

## Difference Between apt update and apt upgrade

### apt update

```bash
sudo apt update
```

This downloads the latest package metadata from the configured Ubuntu repositories.

It tells the server:

> What package versions are currently available?

It normally does not upgrade installed packages.

### apt upgrade

```bash
sudo apt upgrade -y
```

This uses the refreshed package information and installs newer versions of installed packages.

### Easy Way to Remember

```text
apt update
    |
    +---- Refresh package information
              |
              v
apt upgrade
    |
    +---- Install available package updates
```

A common maintenance sequence is:

```bash
sudo apt update
sudo apt upgrade -y
```

---

# 4. Create a Non-Root User

Create a new administrative user:

```bash
sudo adduser devops
```

Add the user to the sudo group:

```bash
sudo usermod -aG sudo devops
```

Verify group membership:

```bash
groups devops
```

The output should include:

```text
sudo
```

Verify the user:

```bash
id devops
```

Switch to the user:

```bash
su - devops
```

Check the current user:

```bash
whoami
```

Expected:

```text
devops
```

Test sudo:

```bash
sudo whoami
```

Expected:

```text
root
```

This proves the user can perform privileged operations through sudo.

---

# 5. SSH Overview

SSH stands for Secure Shell.

It is used to securely access a remote Linux server.

Typical connection:

```bash
ssh devops@SERVER_IP
```

Default SSH port:

```text
22/TCP
```

The two common authentication approaches are:

```text
Password Authentication
        OR
SSH Key Authentication
```

For a hardened server, key-based authentication is preferred.

---

# 6. SSH Key Authentication

An SSH key pair contains:

```text
Private Key
Public Key
```

The private key stays on the local machine.

The public key is installed on the server.

```text
LOCAL MACHINE
+----------------------+
| Private Key          |
| id_ed25519           |
+----------+-----------+
           |
           | Authentication
           v
SERVER
+----------------------+
| Public Key           |
| authorized_keys      |
+----------------------+
```

## Critical Rule

```text
PRIVATE KEY
    -> Never share
    -> Never upload publicly
    -> Keep protected on the client

PUBLIC KEY
    -> Installed on the server
    -> Can be shared for authentication
```

---

# 7. Generate an SSH Key Pair

Run this command on the local machine:

```bash
ssh-keygen -t ed25519
```

Typical files:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

Meaning:

```text
id_ed25519
    = Private Key

id_ed25519.pub
    = Public Key
```

## Why Ed25519?

Ed25519 is a modern public-key algorithm commonly used for SSH authentication.

It provides strong security, compact keys, and good performance.

RSA is also supported if compatibility requires it:

```bash
ssh-keygen -t rsa -b 4096
```

The important concept is:

```text
ed25519 = key algorithm
4096    = RSA key size
```

You do not choose `256`, `512`, etc. for Ed25519 in the same way you choose an RSA bit size.

---

# 8. Are Keys Generated on Two Laptops the Same?

If two laptops independently run:

```bash
ssh-keygen -t ed25519
```

the generated keys will normally be different.

Example:

```text
Laptop A
    |
    +---- Key A

Laptop B
    |
    +---- Key B
```

Therefore:

```text
Key A != Key B
```

The command and algorithm are the same, but cryptographic randomness generates a different key pair.

If the same private key file is copied from Laptop A to Laptop B, then both laptops would be using the same SSH identity.

Best practice is generally to keep separate keys for separate devices and install each device's public key on the server.

---

# 9. Install the Public Key on the Server

If `ssh-copy-id` is available:

```bash
ssh-copy-id devops@SERVER_IP
```

Alternatively, manually create the SSH directory on the server:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Edit:

```bash
nano ~/.ssh/authorized_keys
```

Paste the contents of the local public key:

```text
id_ed25519.pub
```

Set permissions:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Set ownership:

```bash
chown -R devops:devops ~/.ssh
```

---

# 10. Test SSH Key Authentication

Before disabling password authentication, test key-based login from another terminal:

```bash
ssh devops@SERVER_IP
```

Make sure the connection works successfully.

Keep the existing SSH session open while changing SSH configuration.

This is important because if the new key configuration is broken and the old session is closed, you could lock yourself out of the server.

---

# 11. Disable SSH Password Authentication

Edit the SSH server configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Set:

```text
PasswordAuthentication no
```

Ensure public-key authentication is enabled:

```text
PubkeyAuthentication yes
```

Validate the SSH configuration:

```bash
sudo sshd -t
```

If there is no output, the configuration syntax is valid.

Reload SSH:

```bash
sudo systemctl reload ssh
```

Verify the effective configuration:

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|pubkeyauthentication'
```

Expected:

```text
passwordauthentication no
pubkeyauthentication yes
```

Test from a new terminal:

```bash
ssh devops@SERVER_IP
```

---

# 12. Why Disable Password Authentication?

If SSH passwords are enabled, attackers can repeatedly attempt usernames and passwords.

Example:

```text
Internet
    |
    v
SSH Server
    |
    +---- Password Attempt 1
    +---- Password Attempt 2
    +---- Password Attempt 3
    +---- Password Attempt 4
    +---- ...
```

This creates a brute-force risk.

With key-based authentication:

```text
Client Private Key
        |
        v
SSH Authentication
        |
        v
Server Public Key
```

The private key itself is not sent to the server as the authentication secret.

## Interview Answer

> I disabled password-based SSH authentication after verifying key-based login. This reduces the attack surface from password guessing and brute-force attempts.

---

# 13. Configure UFW Firewall

UFW means:

```text
Uncomplicated Firewall
```

It provides a simple interface for managing the Linux firewall.

Check status:

```bash
sudo ufw status
```

Set default incoming policy to deny:

```bash
sudo ufw default deny incoming
```

Allow outgoing traffic:

```bash
sudo ufw default allow outgoing
```

Allow SSH:

```bash
sudo ufw allow 22/tcp
```

Enable UFW:

```bash
sudo ufw enable
```

Check configuration:

```bash
sudo ufw status verbose
```

Expected concept:

```text
Status: active

22/tcp    ALLOW
```

---

# 14. Why Allow Port 22?

SSH normally uses:

```text
TCP 22
```

The administrator needs SSH access for remote server management.

Therefore:

```text
SSH : 22 -> ALLOW
```

Other incoming ports should remain blocked unless specifically required.

---

# 15. Adding Application Ports

When an application is deployed, additional ports may be required.

For HTTPS:

```bash
sudo ufw allow 443/tcp
```

For HTTP:

```bash
sudo ufw allow 80/tcp
```

The security principle is:

> Open only the ports that are actually required.

Example architecture:

```text
Internet
    |
    v
Nginx : 443
    |
    v
Application : 3000
    |
    v
Database : 5432
```

A database such as PostgreSQL should generally not be exposed directly to the public internet. Restrict internal services to trusted networks or hosts.

---

# 16. UFW Default Policy

A hardened baseline can use:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

This means:

```text
Incoming
    -> Deny by default
    -> Allow only required ports

Outgoing
    -> Allow by default
```

This follows the principle:

```text
Default Deny + Explicit Allow
```

---

# 17. Automatic Security Updates

Install unattended-upgrades:

```bash
sudo apt install unattended-upgrades -y
```

Configure it:

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Check the service:

```bash
systemctl status unattended-upgrades
```

The purpose is to automatically apply security updates and reduce the time a server remains exposed to known vulnerabilities.

---

# 18. Fail2Ban

Fail2Ban protects services against repeated brute-force attempts.

Install:

```bash
sudo apt install fail2ban -y
```

Enable and start:

```bash
sudo systemctl enable --now fail2ban
```

Check:

```bash
sudo systemctl status fail2ban
```

Check active jails:

```bash
sudo fail2ban-client status
```

For SSH, depending on the configuration:

```bash
sudo fail2ban-client status sshd
```

---

# 19. How Fail2Ban Works

Simplified flow:

```text
Attacker
    |
    | Repeated failed login attempts
    v
SSH
    |
    v
Authentication Logs
    |
    v
Fail2Ban
    |
    | Detect repeated failures
    v
Temporary IP Ban
```

Fail2Ban monitors logs and reacts to repeated suspicious behavior.

---

# 20. UFW vs Fail2Ban

These are different security layers.

## UFW

UFW is a firewall.

It defines general network access rules.

Example:

```text
Allow TCP 22
Deny other incoming traffic
```

## Fail2Ban

Fail2Ban detects repeated suspicious activity and can dynamically ban offending IP addresses.

Example:

```text
Failed SSH attempts
        |
        v
Fail2Ban detection
        |
        v
Temporary IP ban
```

Therefore:

```text
UFW
    = Network access control

Fail2Ban
    = Dynamic brute-force protection
```

They complement each other.

---

# 21. Configure Timezone

Check current timezone:

```bash
timedatectl
```

For India:

```bash
sudo timedatectl set-timezone Asia/Kolkata
```

Verify:

```bash
timedatectl
```

Correct timezone configuration is important for:

- Log timestamps
- Scheduled jobs
- Monitoring alerts
- Incident investigation
- Cron/systemd timers

---

# 22. Configure Hostname

A meaningful hostname makes infrastructure easier to identify.

Example:

```bash
sudo hostnamectl set-hostname uat-monitoring
```

Verify:

```bash
hostnamectl
```

Examples of meaningful names:

```text
prod-api-01
prod-db-01
uat-monitoring
uat-nginx-01
```

Instead of unclear names such as:

```text
ip-10-0-1-223
```

A meaningful hostname helps administrators quickly identify the server's purpose.

---

# 23. systemctl

Linux services are commonly managed using systemd.

`systemctl` is the command used to control and inspect systemd services.

## Check status

```bash
sudo systemctl status nginx
```

## Start

```bash
sudo systemctl start nginx
```

## Stop

```bash
sudo systemctl stop nginx
```

## Restart

```bash
sudo systemctl restart nginx
```

## Reload configuration

```bash
sudo systemctl reload nginx
```

## Enable at boot

```bash
sudo systemctl enable nginx
```

## Enable and start immediately

```bash
sudo systemctl enable --now nginx
```

---

# 24. start vs enable

This is an important distinction.

### start

```bash
sudo systemctl start nginx
```

Starts the service now.

It does not by itself mean the service will automatically start after reboot.

### enable

```bash
sudo systemctl enable nginx
```

Configures the service to start automatically during boot.

### enable --now

```bash
sudo systemctl enable --now nginx
```

Does both:

```text
Start now
+
Start automatically after reboot
```

---

# 25. restart vs reload

### restart

```bash
sudo systemctl restart nginx
```

The service is stopped and started again.

### reload

```bash
sudo systemctl reload nginx
```

The service reloads its configuration without a complete service restart, when the service supports reload.

For configuration-only changes, reload can reduce disruption.

---

# 26. Check Services

List running services:

```bash
systemctl --type=service --state=running
```

Check whether a service is enabled:

```bash
systemctl is-enabled nginx
```

Check whether it is active:

```bash
systemctl is-active nginx
```

---

# 27. journalctl

`journalctl` is used to inspect logs collected by systemd's journal.

View all journal logs:

```bash
sudo journalctl
```

View the end of the journal:

```bash
sudo journalctl -e
```

View logs for a service:

```bash
sudo journalctl -u nginx
```

Follow service logs in real time:

```bash
sudo journalctl -u nginx -f
```

View current boot logs:

```bash
sudo journalctl -b
```

View the last 100 lines:

```bash
sudo journalctl -n 100
```

View SSH logs:

```bash
sudo journalctl -u ssh
```

---

# 28. /var/log

Traditional Linux logs are commonly stored under:

```text
/var/log/
```

List the directory:

```bash
ls -lah /var/log/
```

Common Ubuntu log files include:

```text
/var/log/syslog
/var/log/auth.log
/var/log/kern.log
```

---

# 29. Authentication Logs

Authentication-related events can be inspected with:

```bash
sudo tail -f /var/log/auth.log
```

These logs can contain events such as:

- SSH login attempts
- Failed authentication
- Successful authentication
- sudo activity

Another useful command:

```bash
sudo journalctl -u ssh -n 50
```

---

# 30. System Logs

View general system activity:

```bash
sudo tail -f /var/log/syslog
```

System logs can help investigate:

- Service failures
- Network events
- Background processes
- System activity
- Errors and warnings

---

# 31. Log Troubleshooting Examples

## SSH login problem

Check:

```bash
sudo journalctl -u ssh -n 50
```

or:

```bash
sudo tail -n 50 /var/log/auth.log
```

Look for messages related to:

```text
Failed password
Authentication failure
Invalid user
Permission denied
Accepted publickey
```

## Service problem

Check:

```bash
sudo systemctl status SERVICE
```

Then:

```bash
sudo journalctl -u SERVICE -n 100
```

For live logs:

```bash
sudo journalctl -u SERVICE -f
```

---

# 32. Security Verification Checklist

## User Setup

```bash
id devops
groups devops
sudo -u devops sudo whoami
```

Expected:

```text
root
```

Checklist:

```text
[ ] Non-root user exists
[ ] User has sudo privileges
[ ] Day-to-day administration uses the non-root user
```

---

## SSH

Check:

```bash
sudo sshd -t
sudo sshd -T | grep -Ei 'passwordauthentication|pubkeyauthentication'
```

Expected:

```text
passwordauthentication no
pubkeyauthentication yes
```

Checklist:

```text
[ ] SSH key generated
[ ] Public key installed
[ ] Key login tested
[ ] Password authentication disabled
[ ] SSH configuration validated
```

---

## Firewall

Check:

```bash
sudo ufw status verbose
```

Checklist:

```text
[ ] UFW enabled
[ ] Default incoming policy is deny
[ ] SSH port 22 allowed
[ ] Only required additional ports are allowed
```

---

## Automatic Updates

Check:

```bash
systemctl status unattended-upgrades
```

Checklist:

```text
[ ] unattended-upgrades installed
[ ] Automatic security updates configured
```

---

## Fail2Ban

Check:

```bash
sudo systemctl status fail2ban
sudo fail2ban-client status
```

Checklist:

```text
[ ] Fail2Ban installed
[ ] Fail2Ban enabled
[ ] Fail2Ban running
[ ] SSH protection configured/verified
```

---

## Hostname

Check:

```bash
hostnamectl
```

Checklist:

```text
[ ] Meaningful hostname configured
```

---

## Timezone

Check:

```bash
timedatectl
```

Checklist:

```text
[ ] Correct timezone configured
[ ] System time synchronized
```

---

## Services

Test:

```bash
sudo systemctl status SERVICE
sudo systemctl start SERVICE
sudo systemctl stop SERVICE
sudo systemctl restart SERVICE
sudo systemctl reload SERVICE
sudo systemctl enable SERVICE
```

Checklist:

```text
[ ] Can check service status
[ ] Can start services
[ ] Can stop services
[ ] Can restart services
[ ] Can reload configuration
[ ] Can enable services at boot
```

---

## Logs

Check:

```bash
sudo journalctl -u ssh -n 50
ls -lah /var/log/
```

Checklist:

```text
[ ] Can inspect systemd journal
[ ] Can inspect authentication logs
[ ] Can locate common /var/log files
```

---

# 33. Final Hardened Server Model

After completing the setup, the server should look conceptually like this:

```text
                         INTERNET
                            |
                            v
                    +---------------+
                    |      UFW      |
                    |   Firewall    |
                    +-------+-------+
                            |
                       Required Ports
                            |
                            v
                    +---------------+
                    |      SSH      |
                    | Key Auth Only |
                    +-------+-------+
                            |
                            v
                    +---------------+
                    | Non-Root User |
                    |   + sudo      |
                    +-------+-------+
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
      Services            Logs            Monitoring
      systemctl         journalctl         /var/log

Additional Security:
--------------------
Fail2Ban
    -> Brute-force protection

unattended-upgrades
    -> Automatic security updates

Hostname
    -> Server identification

Timezone
    -> Correct timestamps
```

---

# 34. Complete Command Reference

```bash
# =========================
# SYSTEM UPDATE
# =========================

sudo apt update
sudo apt upgrade -y


# =========================
# USER SETUP
# =========================

sudo adduser devops
sudo usermod -aG sudo devops

id devops
groups devops

su - devops
whoami
sudo whoami


# =========================
# SSH KEY - RUN ON LOCAL MACHINE
# =========================

ssh-keygen -t ed25519

# Public key:
# ~/.ssh/id_ed25519.pub

# Private key:
# ~/.ssh/id_ed25519


# =========================
# COPY SSH PUBLIC KEY
# =========================

ssh-copy-id devops@SERVER_IP


# =========================
# SSH SERVER CONFIGURATION
# =========================

sudo nano /etc/ssh/sshd_config

# Set:
# PasswordAuthentication no
# PubkeyAuthentication yes

sudo sshd -t
sudo systemctl reload ssh

sudo sshd -T | grep -Ei 'passwordauthentication|pubkeyauthentication'


# =========================
# UFW FIREWALL
# =========================

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose

# Optional application ports:
# sudo ufw allow 80/tcp
# sudo ufw allow 443/tcp


# =========================
# AUTOMATIC SECURITY UPDATES
# =========================

sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades

systemctl status unattended-upgrades


# =========================
# FAIL2BAN
# =========================

sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban

sudo systemctl status fail2ban
sudo fail2ban-client status

# Depending on configuration:
sudo fail2ban-client status sshd


# =========================
# TIMEZONE
# =========================

timedatectl
sudo timedatectl set-timezone Asia/Kolkata
timedatectl


# =========================
# HOSTNAME
# =========================

sudo hostnamectl set-hostname uat-monitoring
hostnamectl


# =========================
# SYSTEMCTL
# =========================

sudo systemctl status SERVICE
sudo systemctl start SERVICE
sudo systemctl stop SERVICE
sudo systemctl restart SERVICE
sudo systemctl reload SERVICE
sudo systemctl enable SERVICE
sudo systemctl enable --now SERVICE

systemctl is-active SERVICE
systemctl is-enabled SERVICE


# =========================
# JOURNALCTL
# =========================

sudo journalctl
sudo journalctl -e
sudo journalctl -u SERVICE
sudo journalctl -u SERVICE -f
sudo journalctl -b
sudo journalctl -n 100
sudo journalctl -u ssh


# =========================
# /var/log
# =========================

ls -lah /var/log/

sudo tail -f /var/log/auth.log
sudo tail -f /var/log/syslog
```

---

# 35. Interview Explanation

If asked to explain this project:

> I worked on hardening a fresh Ubuntu Linux server before application deployment.
>
> First, I created a non-root administrative user and granted it sudo privileges so that day-to-day administration would not be performed directly as root. This follows the principle of least privilege.
>
> I then configured SSH key-based authentication. I generated an Ed25519 key pair locally, installed the public key on the server, tested key-based login, validated the SSH configuration, and disabled password-based SSH authentication.
>
> Next, I configured UFW as the host firewall. I set the default incoming policy to deny and allowed SSH on port 22. The idea is to expose only the ports actually required by applications.
>
> I updated the operating system packages and configured unattended-upgrades to automatically apply security updates.
>
> I installed Fail2Ban to protect SSH against repeated brute-force authentication attempts. It monitors authentication logs and can temporarily ban suspicious IP addresses.
>
> I also configured the server timezone and a meaningful hostname so that logs, scheduled tasks, monitoring alerts, and server identification are easier to manage.
>
> For service management, I used systemctl to check status, start, stop, restart, reload, and enable services at boot.
>
> For troubleshooting and log inspection, I used journalctl and files under /var/log, especially authentication and system logs.
>
> Finally, I verified the user permissions, SSH configuration, firewall rules, automatic updates, Fail2Ban, hostname, timezone, services, and logs.
>
> The overall objective was to take a fresh Ubuntu server and establish a secure baseline before deploying applications.

---

# 36. Key Concepts to Remember

The project can be remembered as:

```text
1. USER
   Create non-root sudo user

2. SSH
   Use key-based authentication
   Disable password authentication

3. FIREWALL
   UFW
   Default deny incoming
   Allow only required ports

4. UPDATES
   apt update
   apt upgrade
   unattended-upgrades

5. BRUTE-FORCE PROTECTION
   Fail2Ban

6. SERVER IDENTITY
   Hostname
   Timezone

7. SERVICES
   systemctl

8. LOGS
   journalctl
   /var/log

9. VERIFY
   Check everything before declaring the server hardened
```

---

# 37. One-Line Summary

> A fresh Ubuntu server was hardened by creating a least-privilege sudo user, configuring SSH key-only authentication, restricting network access with UFW, enabling automatic security updates, protecting SSH with Fail2Ban, configuring hostname/timezone, managing services with systemctl, inspecting logs with journalctl and /var/log, and validating the final security baseline.

---

# Project Completion Checklist

```text
[x] Non-root sudo user created
[x] SSH key-based authentication configured
[x] SSH password authentication disabled
[x] UFW firewall enabled
[x] SSH port 22 allowed
[x] System packages updated
[x] Automatic security updates configured
[x] Fail2Ban installed and enabled
[x] Timezone configured
[x] Meaningful hostname configured
[x] systemctl service management verified
[x] journalctl log inspection verified
[x] /var/log inspection verified
[x] Final security checklist completed
```

## Conclusion

This project establishes a practical security baseline for a fresh Ubuntu Linux server.

The main principles are:

```text
Least Privilege
      +
Key-Based SSH
      +
Firewall
      +
Automatic Security Updates
      +
Brute-Force Protection
      +
Correct Hostname/Timezone
      +
Service Management
      +
Log Inspection
      =
Hardened Linux Server
```
