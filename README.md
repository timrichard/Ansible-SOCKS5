### Ansible playbook for a cloud SOCKS5 server

I wrote this playbook to get the latest version of [microsocks](https://github.com/rofl0r/microsocks) on a cloud VPS.

#### Distro

This playbook was designed to run on Debian (original target was a Debian 10 EC2 instance). It should be easy to adapt for any systemd based distro, as only the apt cache update is specific to Debian (and possibly the build-essential package name).

#### Development helpers

I wrote three utility Bash scripts to manipulate a local VMware VM that's useful for development. I've only tested them on VMware Fusion running on Mac OSX. It's a time saver to snapshot after a section of the playbook is working correctly. That snapshot can be reverted to if necessary, and the idempotent playbook will pass through the tasks that are already working fairly quickly to get to the current task under development that's work in progress. This speeds up development by allowing working tasks to be passed through as 'OK' while still having the ability to revert all changes since the 'good' snapshot for a problematic task.

Run ./capture_vmname to use vmrun to detect the path of the VM in use, and store it in a config file called .vmname

Run ./snapshot to create a snapshot called RTM ('revert to me') of the current state. If a snapshot of that name already exists, it's deleted first. The VM is paused and resumed by the shellscript to let this happen smoothly.

Run ./reset to restore the RTM snapshot. Passing an argument to this script will allow you to restore a snapshot of a different name.

#### Setup

##### Secrets

Use ansible-vault to create a secure yml file. In this case, I use one called secrets.yml
My ansible.cfg is configured to allow ansible-vault to decrypt it using the output of a shellscript called vault-key. This uses the OSX Keychain Access utility to manage the passphrase used to decrypt the vault, which is stored as an application password in the login keychain.

In this case, a passhprase for this project can be set on Mac OSX as follows :

```
security add-generic-password -a "$USER" -s "ansible-socks5-server" -w
```

The shellscript vault-key then retrieves this from the keychain and supplies it to ansible-vault, which allows the secrets inside to be used by the playbook.

Though encrypted, my secrets.yml isn't included as part of this public repo as an extra security measure.

Use ansible-vault to create a secrets.yml file that contains the following keys and values :

```
socks_uname: (your desired username)
socks_pwd: (your desired password)
socks_port: (your desired port)
```

##### ansible.cfg

Apart from directing ansible-vault to the vault-key shellscript for the passphrase for a decode, the config file also sets the inventory of the target server(s) to be a file called 'hosts'.
My hosts file currently contains the SSH config alias of my local VM. Change it to point to any servers desired.

##### Provisioning the target with ansible-playbook

Execute `ansible-playbook socks.yml`

##### Using the SOCKS5 server

I've configured the microsocks systemd service file to use auth_once mode. This means that any successful auth from an IP address results in that IP being on an AllowList kept in memory while the SOCKS5 server is running. It's useful for connecting clients that don't handle the username/password auth correctly. You could, for example, use curl to pre-auth before using any client like a browser that doesn't handle SOCKS auth well. That's useful when you have a dynamic IP address.

```
curl --socks5 (your_socks_username):(your_socks_password)@(remote_socks_host):(socks_port) --silent --show-error --output /dev/null http://google.com
```

That could, for example, be run in an alias along with the command line command to launch Chrome :

```
(path to chrome binary) --proxy-server=socks5://(remote_socks_host):(socks_port)
```
