<!--
Title: How to self-host hardened Borg server
Description: Learn how to self-host hardened Borg server.
Author: Sun Knudsen <https://github.com/sunknudsen>
Contributors: Sun Knudsen <https://github.com/sunknudsen>
Reviewers:
Publication date: 2020-11-27T17:49:18.440Z
Listed: true
-->

# How to self-host hardened Borg server

[![How to self-host hardened Borg server](how-to-self-host-hardened-borg-server.png)](https://www.youtube.com/watch?v=rzEaxL6F2Eg "How to self-host hardened Borg server")

## Requirements

- [Hardened Debian server](../how-to-configure-hardened-debian-server/README.md) or [hardened Raspberry Pi](../how-to-configure-hardened-raspberry-pi/README.md)
- Linux or macOS computer

## Caveats

- When copy/pasting commands that start with `$`, strip out `$` as this character is not part of the command
- When copy/pasting commands that start with `cat << "EOF"`, select all lines at once (from `cat << "EOF"` to `EOF` inclusively) as they are part of the same (single) command

## 1 Setup guide Client  

### 1.1: create `borg` SSH key pair (on computer)

When asked for file in which to save key, enter `borg`.

When asked for passphrase, use output from `openssl rand -base64 24` (and store passphrase in password manager).

```console
$ mkdir ~/.ssh

$ ssh-keygen -t ed25519 -f $HOME/.ssh/borg -C "borg"
Generating public/private ed25519 key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again:
Your identification has been saved in $HOME/.ssh/borg.
Your public key has been saved in $HOME/.ssh/borg.pub.
The key fingerprint is:
SHA256:qQ85VIwDOa3gpJB9mkyi0ERoea8PL0bUqPH4ZkicyuQ Borg
The key's randomart image is:
+--[ED25519 256]--+
| B+ .o           |
|*o*.+..o         |
|=B.=+oo o        |
|o.=o.o o .       |
|. B . . S        |
| B = . o         |
|* + + =          |
|.E * o +         |
|  + .   .        |
+----[SHA256]-----+
```
Public key can be read from `borg.pub`
Refered to as `$client_key_borg_pub`. 
```console
$ cat borg.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHdfSTP05mWqk/JyGGNWb+Af+l+R2xdBz9p2qu5eimWW borg
```

### 1.2: create `borg-append-only` SSH key pair (on computer)

When asked for file in which to save key, enter `borg-append-only`.

When asked for passphrase, leave field empty for no passphrase.

Refered to as `$client_key_borg_append_only`. 
```console
$ ssh-keygen -t ed25519 -f $HOME/.ssh/borg-append-only -C "borg-append-only"
Generating public/private ed25519 key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in $HOME/.ssh/borg-append-only
Your public key has been saved in $HOME/.ssh/borg-append-only.pub
The key fingerprint is:
SHA256:zkdIq1BeZBFmvI9hYfwkoXL/R4wJPC8jcauD1jjWEhw borg-append-only
The key's randomart image is:
+--[ED25519 256]--+
|       oO+       |
|       B* .      |
|    E =.B*       |
|   . * *+B.+     |
|    + o.S+= o    |
|     B *.=..     |
|    B * o o .    |
|   o o . . .     |
|                 |
+----[SHA256]-----+
```

Public key can be read from `borg-append-only.pub`.
Refered to as `$client_key_borg_append_only_pub`. 

```console
$ cat $HOME/.ssh/borg-append-only.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIbm3pulYKSp5p4TzkNM0bYlHn+ByBD8VbNd7EWnap2y borg-append-only
```
### 1.3: generate SSH authorized keys heredoc
To be copyied to Server 

#### Set Borg storage quota environment variable

```shell
BORG_STORAGE_QUOTA="10G"
backup_target='/volum1/@backup/borg-backup'
borg_serve="borg serve --restrict-to-repository $backup_target/$hostname --storage-quota $BORG_STORAGE_QUOTA"
client_key_borg_pub="$HOME/.ssh/borg.pub"
client_key_borg_append_only_pub="$HOME/.ssh/borg-append-only.pub"
```

#### Generate heredoc (the output of following command will be used at [step 2.6](#26-configure-borg-ssh-authorized-keys))

To creat a temperaty file.
```shell
cat << EOF > $HOME/borgbackup.temp.authorized_keys
command="$borg_serve",restrict ${client_key_borg_pub}
command="$borg_serve --append-only",restrict ${client_key_borg_append_only_pub}
EOF
```

#### To copy the file to server.
```shell
scp  $HOME/borgbackup.temp.authorized_keys server-admin@185.112.147.115:/home/server-admin/borgbackup.temp.authorized_keys
```

## 2 Setup guide server

### 2.1: log in to server or Raspberry Pi

> Heads-up: replace `~/.ssh/server` with path to private key and `server-admin@185.112.147.115` with server or Raspberry Pi SSH destination.

```shell
ssh -i ~/.ssh/server server-admin@185.112.147.115
```

### 2.2: switch to root

When asked, enter root password.

```bash
su -
```

### 2.3: update APT index

```bash
apt update
```
### 2.4: create `borg` user

`$backup-target` is your root directory for borg-target example: `/volum1/@backup/borg-backup`

```console
$ adduser --home $backup-target --system borg --shell /bin/bash
Adding system user `borg' (UID 130) ...
Adding new user `borg' (UID 130) with group `nogroup' ...
Creating home directory `/volum1/@backup/borg-backup' ...
```

### 2.5: install [Borg](https://github.com/borgbackup/borg)

```bash
apt install -y borgbackup
```

### 2.6: configure borg SSH authorized keys

#### Create `.ssh` directory

```shell
mkdir -p /home/borg/.ssh
```

#### Create `/home/borg/.ssh/authorized_keys` using heredoc generated at [step 1.3](#13-generate-ssh-authorized-keys-heredoc)

```shell
cat << "_EOF" > /home/borg/.ssh/authorized_keys
command="borg serve --restrict-to-repository /volum1/@backup/borg-backup/client1 --storage-quota 10G",restrict ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHdfSTP05mWqk/JyGGNWb+Af+l+R2xdBz9p2qu5eimWW borg
command="borg serve --restrict-to-repository /volum1/@backup/borg-backup/client1 --storage-quota 10G --append-only",restrict ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIbm3pulYKSp5p4TzkNM0bYlHn+ByBD8VbNd7EWnap2y borg-append-only
_EOF
```

### 2.7 Change ownership and Permissions of `$backup_target`

```shell
chown -R borg:borg $backup_target/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config  
chmod 600 ~/.ssh/authorized_keys 
chmod 600 ~/.ssh/id*
chmod 644 ~/.ssh/*.pub
```

👍
