# Homelab

## K3s Setup on Raspberry Pi(s)

Using the `rpi-imager` tool:
- Select `Raspberry Pi OS 64-bit`
- Select the SD card as storage
- Set the hostname as `rp-i` where `i` is the index
- Enable password-based SSH for a user `pi`
- Configure the wireless LAN (note that Raspberry Pis currently require 2.4GHz channels)

Once written to the SD card, insert and boot the Raspberry Pi. It should boot and be assigned a dynamic IP address on the network. The router's admin console should list the connected devices and their associated IP addresses. With the IP address, you should be able to test the SSH connection by running `ssh pi@$IP` using the password set when imaging the SD card.

### Assign Static IP Addresses
Login to the router's admin console and amend the DHCP range (which may be referred to as a Pool LAN). My range was `192.168.1.64` to `192.168.1.255`. To make space for some static IP addresses, adjust the end value. For instance, `192.168.1.235` provides 20 IP addresses for static assignment.

To set the static addresses:
1. SSH to the Raspberry Pi
   ```shell
   ssh pi@$IP
   ```
2. Install `vim`
   ```shell
   sudo apt-get install vim
   ```
3. Edit `/etc/dhcpcd.conf` (or create it if it does not exist)
   ```shell
   sudo vim /etc/dhcpcd.conf
   ```
4. Add/edit the section for static addresses and add
   ```shell
   interface eth0
   static ip_address=192.168.1.244/24
   static routers=192.168.1.1
   static domain_name_servers=192.168.1.1
   ```

   In this case, set `192.168.1.244` to the static IP which you intend to set for the Raspberry Pi. Also, set `192.168.1.1` to the router's IP
5. Reboot the Raspberry Pi
   ```shell
   sudo reboot
   ```
6. After reboot, SSH to the static IP
   ```shell
   ssh pi@192.168.1.244
   ```
7. If this does not work, assign a static lease via the network router's admin console. Less elegant, but sure to work.
8. Setup `/etc/host` mappings on the local machine for each Raspberry Pi
   ```shell
   192.168.1.244 rpi-1
   ...
   192.168.1.xxx rpi-n
   ```

### Setup Password-less SSH (Optional)
This is just a quality of life option to prevent having to retype passwords when connecting to Raspberry Pis

1. Generate an SSH key on your local machine
   ```shell
   ssh-keygen
   ```
   Accept default options and don't enter a password. This will create a `~/.ssh/id_rsa` (private key) and `~/.ssh/id_rsa.pub` (public key)
2. Copy the contents of `~/.ssh/id_rsa.pub`
3. SSH to a Raspberry Pi and run:
   ```shell
   mkdir ~/.ssh
   touch ~/.ssh/authorized_keys
   chmod 0700 ~/.ssh
   chmod 0600 ~/.ssh/authorized_keys
   vim ~/.ssh/authorized_keys
   ```
   And paste the contents of the public key. Save the file
4. Repeat this for each Raspberry Pi

After this, you should be able to connect to any Raspberry Pi without entering a password

### Install K3s
K3s can be installed via the [k3s-ansible](https://github.com/k3s-io/k3s-ansible/tree/master) setup, which requires Ansible. Assuming you have Python3 installed:

```shell
python3 -m venv .venv
source .venv/bin/activate
pip3 install ansible
```

Now to install K3s:
1. Clone the [k3s-ansible](https://github.com/k3s-io/k3s-ansible/tree/master)
2. Create the inventory by copying the example:
   ```shell
   cp inventory-sample.yml inventory.yml
   ```
3. Edit the `inventory.yml` to match the cluster setup. For instance, if we have the following three Raspberry Pi devices configured:
   ```shell
   192.168.1.244
   192.168.1.245
   192.168.1.246
   ```

   We could configure the cluster as follows:
   ```yml
    k3s_cluster:
        children:
            server:
                hosts:
                    192.168.1.244:
            agent:
                hosts:
                    192.168.1.245:
                    192.168.1.246:
   ```

   The K3s version can also be configured. Also set the `ansible_user` to the `pi` which was set when imaging the SD card(s):
   ```shell
   vars:
     ansible_user: pi
     k3s_version: v1.26.9+k3s1
   ```
4. Run the Ansible playbook
   ```shell
   ansible-playbook playbook/site.yml -i inventory.yml
   ```

### Configure Kubectl
It may take a couple of runs of the playbook for it to successfully install. Once completed, `kubectl` can be configured on the local machine to connect to K3s

1. Transfer the `kubeconfig` from the master node to the local machine
   ```shell
   scp pi@rpi-1:~/.kube/config ~/.kube/config-rpi-k3s
   ```
2. Configure the `KUBECONFIG` environment variable (add to `~/.bashrc`)
   ```shell
   export KUBECONFIG="/home/$USER/.kube/config"
   export KUBECONFIG="$KUBECONFIG:/home/$USER/.kube/config-rpi-k3s"
   ```

   On my machine, the `KUBECONFIG` requires absolute paths rather than relative
3. Use `kubectl` to check the available config contexts
   ```shell
   kubectl config get-contexts
   ```

   Ensure the Raspberry Pi K3s context has a useful name. If not, change it with:
   ```shell
   kubectl config rename-context $CURRENT_NAME $NEW_NAME
   ```
4. Use a tool like [`kubectx`](https://github.com/ahmetb/kubectx) to readily switch between clusters and view the nodes in the `rpi-k3s` cluster
   ```shell
   kubectl get nodes
   ```
