# Kolla Builder

This repository contains a set of Ansible playbooks that will spawn and configure VMs (currently supporting libvirt only) to deploy Kolla.

## Prerequisites

- Libvirt
  - Including `libvirt-dev`
  ```bash
  sudo apt-get install libvirt-dev
  ``` 
- Virt-manager (optional)
- [Kolla image](#) - TODO: Upload the Kolla image somewhere
- 3 NAT networks - [Guide](https://gulraezgulshan.medium.com/virtual-networking-in-linux-b1abcb983e72)
  - **nat1:** `192.168.122.0/24` for SSH and Ansible
  - **nat2:** `192.168.123.0/24` for OpenStack management network
  - **nat3:** `192.168.100.0/24` for Neutron networking
  ```bash
  virsh net-create libvirt/nat1.xml
  virsh net-create libvirt/nat2.xml
  virsh net-create libvirt/nat3.xml
  ```
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
## Configuration

- Edit the example user config and save it as `user_config.yml` (options are documented in config examples)
  - For All in one deployment adjust the [example_aio_user_config.yml](example_aio_user_config.yml) example config
  - For Multinode deployment adjust the [example_multinode_user_config.yml](example_multinode_user_config.yml) example config
- Edit [kolla-files/globals.yml](kolla-files/globals.yml)
    - This is the configuration file of Kolla - [Read the Docs](https://docs.openstack.org/kolla-ansible/latest/admin/index.html)
    - One of the most important options is `kolla_internal_vip_address`, which should be any *NOT USED* address in the **nat2** range, e.g.,
    ```yaml
    kolla_internal_vip_address: 192.168.123.200 # The default
    ```

## Usage

### Spawn nodes

#### All in one

- Spawn the node

```bash
ansible-playbook -i local spawn-aio.yml -e @user_config.yml
```

- Prepare the node

```bash
ansible-playbook -i all-in-one prepare.yml -e @user_config.yml
```

- SSH into the node

```bash
ssh <node-ip> -o "IdentitiesOnly=yes" -i ~/.ssh/id_kolla
```

- Deploy using the deploy script

```bash
./deploy
```

- If it fails, please refer to [Kolla documentation](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html#install-ansible-galaxy-requirements) and follow from the linked point.
- It may fail due to a network issue; in that case, just run it again.

#### Multinode

- Spawn your VMs running this playbook

```bash
ansible-playbook -i local spawn-multinode.yml -e @user_config.yml
```

- Check the IPs on the console output and in the generated `multinode` inventory file. If you see `<NODE_NAME>_IP_NOT_FOUND`, then find the IP manually (through virt-manager). It might be that the node didn't start for any reason or it didn't boot fast enough before the playbook gave up retrieving IP.

- If you need to log into a node without SSH, use these credentials

```
username: kolla
password: hhh
```

- Check **nat2** IPs of the nodes and set `kolla_internal_vip_address` in [globals.yml](kolla-files/globals.yml) to an unused IP in `192.168.123.0/24` range.

### Prepare nodes

- Prepare your nodes using this playbook

```bash
ansible-playbook -i multinode prepare.yml -e @user_config.yml
```

- SSH into your deployment node

```bash
ssh <deployment-node-ip> -o "IdentitiesOnly=yes" -i ~/.ssh/id_kolla
```

### Deploy script

- You can try the deploy script which is already on the node

```bash
./deploy
```

- If that fails, investigate the issue. Sometimes you just have to run the script again. If it doesn't help, see the next section.

### Manual deployment

Based on [Quick start for development](https://docs.openstack.org/kolla-ansible/latest/user/quickstart-development.html)

- A virtual environment for Kolla is sourced automatically on SSH into the node.
- A copy of Kolla Ansible is already present.

#### Pre-deploy

These steps can be done automatically by `./pre-deploy` script as well.

- Install Kolla dependencies

```bash
kolla-ansible install-deps
```

- Known issue: sometimes `ansible-galaxy` commands fail due to network issues. Just run the command again.

- Generate certificates in case you use TLS

```bash
kolla-ansible -i multinode certificates
```

- Bootstrap servers

```bash
kolla-ansible -i multinode bootstrap-servers
```

- Run prechecks

```bash
kolla-ansible -i multinode prechecks
```

#### Deployment

- Run

```bash
kolla-ansible -i multinode deploy
```

- If the command fails, you may have issues with Kolla (especially if you changed the playbooks) or your nodes might not have enough resources.
- You also might check your globals file, especially `network_interface` and `neutron_external_interface` must be existing interfaces on all of your nodes.
Check using `ip a` command. However, the node [XML](roles/create_vm/templates/kolla-node.xml.j2) is configured in a way that the interface names should always be
`network_interface: enp6s0` and `neutron_external_interface: enp7s0`.
- You might have a sort of bad luck that one of the nodes gets the same IP as `kolla_internal_vip_address`. In that case, change `kolla_internal_vip_address`.

#### Post-deploy

- Install OpenStack client

```bash
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/master
```

- Run post-deploy jobs

```bash
kolla-ansible post-deploy
```

- Create example resources

```bash
./init-runonce
```

### Deletion

- Delete all nodes using

```bash
ansible-playbook -i local delete.yml -e @vm_list.yml -e @user_config
```
