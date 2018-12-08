# SSH Essentials

<!--ts-->
   * [SSH Essentials](#ssh-essentials)
      * [Overview](#overview)
      * [Authentication](#authentication)
         * [How SSH works](#how-ssh-works)
         * [Generating SSH keys](#generating-ssh-keys)
      * [Copying your Public SSH Key to a Server](#copying-your-public-ssh-key-to-a-server)
         * [1. Using scp](#1-using-scp)
         * [2. Using ssh-copy-id](#2-using-ssh-copy-id)
      * [Security](#security)
         * [Disable password authentication](#disable-password-authentication)
         * [Changing the port that SSH Daemon runs on](#changing-the-port-that-ssh-daemon-runs-on)
         * [Limit number of users who can connect through SSH](#limit-number-of-users-who-can-connect-through-ssh)
         * [Disable Root login](#disable-root-login)
         * [Allow Root access for specific commands](#allow-root-access-for-specific-commands)
         * [Allowing GUI](#allowing-gui)
      * [Client-Side configuration options](#client-side-configuration-options)
         * [Defining Server-Specific Connection Information](#defining-server-specific-connection-information)
         * [Disabling Host Checking](#disabling-host-checking)
   * [References](#references)

<!-- Added by: crysis, at: 2018-12-08T23:55+05:30 -->

<!--te-->

## Overview

- For the duration of your SSH session, any commands that you type into your local terminal are **sent through an encrypted SSH tunnel** and executed on your server.
- client-server model
- **server** should run **SSH Daemon**
- **you** must have **SSH Client**

## Authentication
Two ways:
1. Passwords (less secure)
2. SSH keys (very secure)

**SSH keys**
- Public key (shared with public)
- Private key (vigilantly guarded)
---

### How SSH works

1. The server(the machine you want to connect to) has list of authorized keys in `~/.ssh/authorized_keys` file. Your public key must be present in this `authorized_keys` file of the server.

2. When you try to connect to the server, you tell that machine which public key it should refer to.

3. The server will then encrypt a random string with your public key and send to you.

4. You decrypt this string and combine with previously negotiated session ID. Generate MD5 hash of this value. Send back to server.

5. Server, who already has session ID and string, compares this hash values to confirm that client is legitimate.

### Generating SSH keys

Create a key pair
```bash
ssh-keygen
```
`ssh-keygen` can generate RSA key pairs, by default key pairs are stored as ` ~/.ssh/id_rsa` (DO NOT SHARE THIS FILE!) and ` ~/.ssh/id_rsa.pub` here, `.pub` is for public key whose content is added to `~/.ssh/authorized_keys` on all machines where the user wishes to log in using public key authentication.

SSH keys are 2048 bits by default. Generate longer or shorter keys using `-b` argument.
```bash
ssh-keygen -b 4096
```

---

## Copying your Public SSH Key to a Server

1. Using `scp`.
2. Using `ssh-copy-id`

### 1. Using `scp`
Generate keys
```bash
ssh-keygen -t rsa
```
Create `.ssh` directory on server and copying your public key to server's `authorized_keys` file
```bash
cat ~/.ssh/id_rsa.pub | ssh root@192.168.1.3 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
Login
```bash
ssh root@192.168.1.3
```

### 2. Using `ssh-copy-id`
Avoid all the above hussle with one command.
```bash
ssh-copy-id root@192.168.1.3
```

## Security

### Disable password authentication

Once ssh keys are configured, proceed to disable password authentication.
On server side we have to implement the following
```bash
sudo vim /etc/ssh/sshd_config
```
Search for `PasswordAuthentication` directive and uncomment it.
```vim
PasswordAuthentication no
```
close the file and restart the ssh service.

### Changing the port that SSH Daemon runs on
```bash
sudo vim /etc/ssh/sshd_config
```
Find `Port 22` specification and modify it to a higher port.
```vim
#Port 22
Port 4444
```
Save and restart the service.

On the **client side**:
Use `-p` to connect
```bash
ssh -p 4444 nitish@192.168.1.3
```

### Limit number of users who can connect through SSH
```bash
sudo vim /etc/ssh/sshd_config
```
Search for the `AllowUsers` directive and if it's not there then create one.
```vim
AllowUsers user1 user2
```
```vim
AllowGroups sshmembers
```
Add users to `sshmembers` group
```bash
sudo usermod -a -G sshmembers user1
sudo usermod -a -G sshmembers user2
```
Restart ssh daemon.

### Disable Root login
```bash
sudo vim /etc/ssh/sshd_config
```
Inside, search for a directive called `PermitRootLogin`. If it is commented, uncomment it. Change the value to `no`:
```vim
PermitRootLogin no
```

### Allow Root access for specific commands

Make a separate key for each command and put it into root user's `authorized_keys` file on the **server**.

```bash
ssh-copy-id root@192.168.1.3
```
Log into server and edit the key file
```bash
sudo vim /root/.ssh/authorized_keys
```
At the beginning of the line with the key you uploaded, add a `command=` listing that defines the command that this key is valid for. This should include the full path to the executable, plus any arguments:
```vim
command="/path/to/command arg1 arg2" ssh-rsa ...
```
**Note:** use `which` to find exact path of the command.
Now, move on to `sshd_config` file.
```bash
sudo vim /etc/ssh/sshd_config
```
Find the directive `PermitRootLogin`, and change the value to `forced-commands-only`.

```vimhttps://github.com/w4rb0y
PermitRootLogin forced-commands-only
```

### Allowing GUI
The SSH daemon can be configured to automatically forward the display of X applications on the server to the client machine.

```bash
sudo vim /etc/ssh/sshd_config
```
Make
```vim
X11Forwarding yes
```
On the **client side**:
Use `-X` to connect
```bash
ssh -X nitish@192.168.1.3
```

## Client-Side configuration options

### Defining Server-Specific Connection Information

We can define individual configurations for some or all of the servers we connect to.These can be stored in the `~/.ssh/config` file, which is read by our SSH client each time it is called.

```bash
vim ~/.ssh/config
```
Inside, we can define individual configuration options by introducing each with a `Host` keyword, followed by an _alias_. Beneath this and indented, we can define any of the directives found in the ssh_config man page `man ssh_config`.
```vim
Host testhost
    HostName example.com
    Port 4444
    User demo
```
You could then connect to `example.com` on port `4444` using the username `demo` by simply typing:
```bash
ssh testhost
```

We can also use wildcards to match more than one host. Keep in mind that **later matches can override earlier ones**.

For instance,
```vim
Host *
    ForwardX11 no

Host testhost
    HostName example.com
    ForwardX11 yes
    Port 4444
    User demo
```
We could default all connections to not allow X forwarding, with an override for example.com by having these entries in our file.

### Disabling Host Checking
```vim
The authenticity of host '111.111.11.111 (111.111.11.111)' can't be established.
ECDSA key fingerprint is fd:fd:d4:f9:77:fe:73:84:e1:55:00:ad:d6:6d:22:fe.
Are you sure you want to continue connecting (yes/no)? yes
```
This can be disabled by setting `StrictHostKeyChecking` as `no`.
```bash
vim ~/.ssh/config
```
```vim
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

# References

Source of these notes: [SSH Essentials](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys)  
[sshd config file manual](https://www.ssh.com/ssh/config/)
