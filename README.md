# Kolla Builder

This repository contains a set of Ansible playbooks that will spawn and configure VMs (currently supporting libvirt only) to deploy Kolla.

## Prerequisites

- Libvirt
  - Including `libvirt-dev`
  ```bash
  sudo apt install qemu qemu-kvm libvirt-dev
  ```
- Virt-manager (optional)
  ```bash
  sudo apt install virt-manager
  ```
- [Kolla image](https://api.gx-scs.sovereignit.cloud:8080/swift/v1/AUTH_0b3c75f80b6743778daccec0da423465/Kolla%20Builder%20Image%2020240903/kolla-image.qcow2)
- 3 NAT networks - [Guide](https://gulraezgulshan.medium.com/virtual-networking-in-linux-b1abcb983e72)
  - **nat1:** `192.168.122.0/24` for SSH and Ansible
  - **nat2:** `192.168.123.0/24` for OpenStack management network
  - **nat3:** `192.168.100.0/24` for Neutron networking
  ```bash
  virsh net-define libvirt/nat1.xml && virsh net-start nat1
  virsh net-define libvirt/nat2.xml && virsh net-start nat2
  virsh net-define libvirt/nat3.xml && virsh net-start nat3
  ```
  - You can use libvirt `default` instead of one of the NATs
- Ansible
  ```bash
  pip install ansible
  ```
- [Kolla Ansible source code](https://github.com/openstack/kolla-ansible)
  ```bash
  git clone https://opendev.org/openstack/kolla-ansible.git
  ```
- Copy SSH key from `ssh/id_kolla` to your `~/.ssh` directory
  ```bash
  cp ssh/id_kolla ~/.ssh
  ```
- Python libraries:
```bash
pip install libvirt-python
pip install lxml
```
- Docker for ARA (optional) - for ARA recording Ansible docker needs to be installed and runnig
## Configuration

- Edit the [example user config](example_config.yml) and options are documented in config examples.

- Edit [kolla-files/globals.yml](kolla-files/globals.yml)
    - This is the configuration file of Kolla - [Read the Docs](https://docs.openstack.org/kolla-ansible/latest/admin/index.html)
    - One of the most important options is `kolla_internal_vip_address`, which should be any *NOT USED* address in the **nat2** range, e.g.,
    ```yaml
    kolla_internal_vip_address: 192.168.123.200 # The default
    ```

## Usage

### Building Environment
The `builder` script is used to manage nodes with `ansible-playbook` .
#### Deploying ARA records ansible server

To deploy ARA server and start recording host ansible plays with ARA,
ensure flag `ara_enable` in `user_config.yml` is set to true and run:

```bash
./builder ara user/local-aio.yml
```

Nodes plays will be recorded to this server too. ARA UI will then be accessible
on localhost/remote host on specified `ara_server_port`, 8000 by default.


#### Spawning Nodes

To spawn a node or nodes, run:

```bash
./builder spawn user/local-aio.yml
```

If you want to spawn the nodes on a remote server, use the `-r` option:

```bash
./builder -r spawn user/remote-multinode.yml
```

> **Note**: Only run the `spawn` action once to create the nodes. You need to delete them before
running again.

#### Preparing Nodes

Once the nodes are booted, prepare them by running:

```bash
./builder prepare user/local-aio.yml
```

If you need to update something on the nodes, run the prepare action again:

```bash
./builder prepare user/local-aio.yml
```

#### Accessing Nodes

- For aio deployment
```bash
ssh -F ssh_config openstack-aio
# or whatever you named your node
```

- For multinode deployment
```bash
ssh -F ssh_config openstack-deployment
# or whatever you named your deploy node
```
#### Configuring Reverse Proxy on Remote Nodes

If you are using the remote option and need to configure a reverse proxy, run:

```bash
./builder -r nginx user/remote-aio.yml
```

### Deploy script

- You can try the deploy script which is already on the node

```bash
./deploy
```

- If that fails, investigate the issue. Sometimes you just have to run the script again. If it doesn't help, see the next section.

- Otherwise reffer to [Quick start for development](https://docs.openstack.org/kolla-ansible/latest/user/quickstart-development.html)

### Remote Setup
- You can use kolla-builder to spawn Kolla on a remote server/cloud instance
- Edit [remote](remote) inventory and add your server
```
PUBLIC.IP.OF.REMOTE ansible_ssh_user=ubuntu ansible_become=True ansible_private_key_file=/path/to/your/ssh_key
```
- If you use a remote Openstack instance, you need:
  - Network setup that allows outside connection
  - SSH keypair
  - Floating IP
  - Security group rules allow:
    - Egress for all connections
    - Ingress for SSH(22)
    - Ingress for HTTP(80)/HTTPS(443) to access Horizon from local browser
    - Ingress for any additional port you reverse proxy to
  - All the [prerequisites](#prerequisites) must be installed on the instance

### Deletion


To delete all nodes, use the delete action:

```bash
./builder delete user/local-multinode.yml
```
or for remote libvirt host

```bash
./builder -r delete user/local-multinode.yml
```
