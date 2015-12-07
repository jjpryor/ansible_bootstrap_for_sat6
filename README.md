# bootstrap_for_sat6
Ansible playbooks for installing Satellite 6.1 on RHEL7.x on target node.
You'll need a mnachine to run the ansible. This was tested with Ansible 1.9+ on
Fedora 22.

The ["Red Hat Satellite Installation Guide"](https://access.redhat.com/documentation/en-US/Red_Hat_Satellite/6.1/html-single/Installation_Guide/index.html#sect-Red_Hat_Satellite-Installation_Guide-Installing_Red_Hat_Satellite_Server-Obtaining_the_Required_Packages)
is the reference resource from which this playbook is composed.


# Start here with remote account pre-reqs
git clone this repo to the a working area in your `$HOME`.

On the Satellite node, perform the following steps to enable the remote user and
the remote user's public SSH key placed into the `.ssh/authorized_keys` file.

SSH into the target node:
 ```shell
 ssh root@targetnode
 ```

Now as root, add the nonprivuser:
```shell
useradd nonprivuser -u SomeLargeNumberToPreventUidCollision
```

Add `nonprivuser` to /etc/group `wheel`: `usermod -G wheel -a nonprivuser`

Change the `sudoers` file to have the wheel group able to run sudo commands
without requiring a password. *This is obviously weakens security, and should
be removed later*
```
%wheel  ALL=(ALL)       NOPASSWD: ALL
```

Now become the `nonprivuser`:
```shell
su - nonprivuser
```

As nonprivuser, make the .ssh directory, set the correct permissions, and
copy/paste the `authorized_keys` file into place, then exit the `nonprivuser`:
```shell
cd
mkdir .ssh
chmod 600 .ssh
vim .ssh/authorized_keys
<paste the public key>
chmod 600 .ssh/authorized_keys
exit
```

From your desktop, enable `ssh-agent` if your desktop environment doesn't
automatically have it running, and also add your ssh identity to the agent.

(*Quite possibly the wrong way to do this, but it works*)
```shell
exec ssh-agent bash
ssh-add ~/.ssh/your_private_key
```
Then check if ssh nonprivuser@targetnode via key & passphrase works and also
test if password-less sudo works:
```shell
ssh -i ~/.ssh/your_private_key nonprivuser@targetnode
sudo touch /etc/foobar
```

Now test ansible to see if it will work:
```
ansible -m setup satellite-node.yourorg.local
```
It should connect via SSH and gather facts of the target
and display the facts in the console, something like:
```
satellite-node.yourorg.local | success >> {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "111.222.323.423
        ],
        "ansible_all_ipv6_addresses": [
            "fe80::250:6bff:fe89:689f"
        ],
        "ansible_architecture": "x86_64",
        "ansible_bios_date": "04/14/2014",
        "ansible_bios_version": "6.00",
        "ansible_cmdline": {
            "BOOT_IMAGE": "/vmlinuz-3.10.0-229.el7.x86_64",
            "crashkernel": "auto",
            "nofb": true,
            "rd.lvm.lv": "sysvg/root",
            "ro": true,
            "root": "/dev/mapper/sysvg-root"
        },
```
## Hardware & Disk space requirements
Just about ready to run the ansible playbooks, but we need to check
if there is not enough space in the filesystem, then use `lvresize` to resize
logical volume of the partition.
Examples:
```shell
lvresize -r -L 110G /dev/mapper/varvg-var
lvresize -r -L 5G /dev/mapper/sysvg-tmp
```
Consider the following to be desirable minimum HW specs

| Node | Cores | RAM | /opt/ | /var/ | /tmp/ |
| --- | --- |  --- |  --- |  --- |  --- |
|Satellite node|2|8 GB|2 GB|100 GB| 5G free|

# Execute the ansible playbooks
If it is all successful, then it is time to start using the ansible playbooks.
They are designed to get all the pieces in place and check to see if the disk
space requirements are met.

```Shell
ansible-playbook 01-prereqs.yml
ansible-playbook 02-bootstrap-sat6.yml
ansible-playbook 03-install-sat6.yml
```
It will install `katello` aka Satellite and will then run the installer.

---

## Contents of sat6 directory
### ansible.cfg
I'm pointing ansible to use my SSH private key and I'm using my `jjpryortemp`
remote account that must be manually setup on each of the three nodes.

### ansible_hosts
It should be obvious that this is where you define the host that these playbooks
will be interacting with.
```
[sat6-node]
satellite-node.yourorg.local
```

### 01-prereqs.yml
+ Performs `yum update -y`
+ Enables `$proxy_*tp` bash shell environment variables
+ Disables & stops `firewalld`
+ Enables & starts `iptables`
+ Copies `ntp.conf` and restarts `ntpd`
+ Copies `iptables` and restarts `iptables`
+ Checks `/var` & `/tmp` for minimum size for Satellite.

### 02-bootstrap-sat6.yml
+ Sets the following environment variables:
  + `ftp_proxy`
  + `http_proxy`
  + `https_proxy`
  + `no_proxy`
+ Prompts the user for the Upstream RHN username and password
+ Checks if `systemid` exists, and if so moves it to `/root`
+ Register the node with Upstream RHNRegister the node with Upstream RHN
+ Subscribe to RHEL7 pool and the Sat6 Pool
+ Disables existing YUM repos, and adds the needed YUM repos
+ Installs `katello` aka Satellite.

### 03-install-sat6.yml
+ Sets the following environment variables:
  + `ftp_proxy`
  + `http_proxy`
  + `https_proxy`
  + `no_proxy`
+ Prompts the user for the Org Name, Location, Satellite Admin Username & password
+ Runs the installer and enables outbound http proxy for Satellite
+ Enable RH subscription-manager outbound proxy

-----
## Contents of files directory

### files/proxy.sh
Shell environment variables for the proxy & this goes into `/etc/profile.d`

### files/wgetrc
`wget` config file & this goes into `/etc/`

### files/iptables-before-install
`iptables` config for Satellite to be put in place before install

### files/iptables-after-install
`iptables` config for Satellite to be put in place after install

### ntp.conf
NTP time config
