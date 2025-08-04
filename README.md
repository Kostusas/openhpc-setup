# OpenHPC Cluster Deployment with Ansible

This guide provides instructions to deploy an OpenHPC cluster using Ansible. Follow the steps below in the order they are presented.

## Prerequisites

Control node:
- Rocky Linux 9.4 on the head node
- Git, Python, pip, Ansible and its collections:
  ```bash
      dnf install git python3.9-pip -y
      pip install --user ansible
      ansible-galaxy install -r requirements.yml
      pip install -r requirements.txt
  ```
- Head node has at least one interface with internet access (VLAN 1713 Core for example)
- Head node has at least one interface connected to the LAN network (VXLAN for example)

Worker nodes:
- PXE boot ready through the LAN network headnode is connected to
- Worker nodes are connected to the same LAN network as the head node (VXLAN for example)

For both devices, make sure the NIC being emulated is "rtl8139" (or just not "virtio") in the VM settings.

## Roles

There are following roles in the `roles` directory:

- `openhpc_install`: Installs core OpenHPC, Warewulf and Slurm packages.
- `openhpc_init`: Configures the head node services for OpenHPC using Ansible tasks based on the OpenHPC installation guide.
- `firewall`: Configures the firewall on the head node to allow PXE boot and other services.
- `slurmctl_init`: Initializes the Slurm controller on the head node.
- `grafana`: Installs and configures Grafana with pre-configured dashboards.
- `mysql`: Installs and configures MariaDB for Slurm accounting.
- `slurm_export`: Slurm data exporter to Prometheus.

## Inventory Setup

Define your inventory in `inventory.ini`.
Use `control` for the head node and `compute` for the worker nodes.

```
[control]
localhost ansible_connection=local

[compute]
c1 ansible_host=10.0.1.1 mac=56:69:bd:26:4d:02
c2 ansible_host=10.0.1.2 mac=56:69:bd:26:4d:03
```

## Host Variables

Make sure to configure your `localhost` host variables in `host_vars/localhost.yml` according to your system configuration.

## Role Variables

In addition to host variables, it's also optional to adjust the default role variables in `roles/<role>/defaults/main.yml`:

## Playbooks and the order of execution

### Base Software Installation

To preload a Rocky 9 image with all required OpenHPC software, run:

```sh
ansible-playbook playbooks/hpc_install.yml
```

### HPC Cluster Initialization

After the packages are installed, configure the head node services with:

```sh
ansible-playbook playbooks/hpc_init.yml
```

### IMPORTANT Pulling Compute Image

Before adding compute nodes, you need to run the following commands:

```bash
dnf install -y apptainer
apptainer build --docker-login --sandbox /tmp/rocky-compute/ docker://git.labs.vu.nl:5050/itvo/proto-openhpc:latest
```

where you will be prompted to login with your GitLab username/password.
Once completed, you will obtain a `.sif` file which contains the image for the compute nodes.
After that, set them up for warewulf:

```bash
wwctl container import /tmp/rocky-compute/ rocky-compute --syncuser
wwctl container build rocky-compute
```

In the future, this will all become part of the playbook.

### Add Compute Nodes

To add compute nodes to the Warewulf database, run the following playbook:

```sh
ansible-playbook playbooks/add_nodes.yml -i inventory.ini
```

At this point, you should restart your compute nodes manually on Open Nebula.
After the restart, the compute nodes should be able to PXE boot and join the cluster.

### Set-up Slurm

Once the compute nodes have fully booted, you need to start the Slurm daemon on the head and compute nodes.

```sh
ansible-playbook playbooks/slurm_init.yml -i inventory.ini
```

### Set-up monitoring

To set up monitoring with Grafana and Prometheus, run the following playbook:

```sh
ansible-playbook playbooks/monitoring.yml -i inventory.ini
```

Monitoring compute nodes is broken for now.
