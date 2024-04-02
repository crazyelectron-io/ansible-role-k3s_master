# role-k3s_master

Ansible role to deploy K3s on master nodes

> Be aware that this role is highly opinionated and fits my preferences and way of working.
> It may or may not be suitable for your needs.

The hosts to target must be in a group named 'master', e.g.:

```yaml
# file: inventories/dev/hosts.yml
# synopsis: inventory file for development environment
---
all:
  children:
    master:
      hosts:
        kube-m01:
          ansible_host: 10.100.3.211
        kube-m02:
          ansible_host: 10.100.3.212
```

## Role variables

### Mandatory variables

`metal_lb_ip_range` _must_ be defined to specify the IP addresses that can be used by MetalLB for loadbalancer entrypoints.

`apiserver_endpoint` - The virtual IP address which will be configured on each master.

`k3s_token` - Required fot masters to talk together securely (this token must be alphanumeric).

`shell_user` _must_ be defined for user-based configuration (like ZSH, Locale, etc.).

These variables can be specified in the `hosts.yaml` inventory file, in the `group_vars`, or in the `vars`section of an Ansible playbook, like this:

```yaml
- hosts: all
  become: true
  gather_facts: true
  vars:
    metal_lb_ip_range: "10.20.30.40-10.20.30.50"
    apiserver_endpoint: "10.10.10.10"
    k3s_token: "th1sisaSupesecretToken"
  roles:
    ...
    - k3s-master
    ...
```

An `hosts.yaml` inventory file will look like this:

```yaml
---
all:
  vars:
    - metal_lb_ip_range: "10.20.30.40-10.20.30.50"
    - apiserver_endpoint: "10.10.10.10"
    ...
  children:
  ...
```

### Important optional parameters (with defaults)

`extra_server_args: "--disable servicelb --disable traefik --tls-san {{ apiserver_endpoint }} --etcd-expose-metrics true --kube-proxy-arg metrics-bind-address=0.0.0.0 --kube-scheduler-arg bind-address=0.0.0.0  --kube-apiserver-arg default-not-ready-toleration-seconds=10 --kube-apiserver-arg default-unreachable-toleration-seconds=10 --kube-apiserver-arg=feature-gates=MixedProtocolLBService=true --kubelet-arg containerd=/run/k3s/containerd/containerd.sock --kubelet-arg node-status-update-frequency=5s --kubelet-arg shutdownGracePeriod=30s --kubelet-arg shutdownGracePeriodCriticalPods=10s --kube-controller-arg node-monitor-grace-period=10s --kube-controller-arg pod-eviction-timeout=60s --kube-controller-arg node-monitor-period=10s --kube-controller-manager-arg bind-address=0.0.0.0"`

The above defines the commandline parameters for K3s server components. Change these to your liking, the only required ones are --disable servicelb --disable traefik, because we use MetalLB and deploy Traefik separately with a different configuration.

`flannel_iface: bond100` - Defines the network interface which will be used for flannel.

`public_dns: 1.1.1.1 8.8.8.8` - Our public DNS servers, starting with Cloudflare.

`kube_vip_tag_version: "v0.5.0"` - Image tag for kube-vip.

`metal_lb_speaker_tag_version: "v0.13.4"` - Image tag for metalLB.
`metal_lb_controller_tag_version: "v0.13.4"`

## Usage of this role

To use this role, include the following section in a `requirements.yml` file in the local `roles` directory:

```yaml
# Include the 'debian-base` role from GitHub
- src: git@github.com:crazyelectron-io/role-k3s_master.git
  scm: git
  version: main
  name: k3s-master
```

> Only include the 'top' roles, dependencies - when listed in `meta/main.yml` of the imported role - will be downloaded automatically.

To retrieve roles like this in your project, run `ansible-galaxy install -r roles/requirements.yml`.
Because these roles will not be updated locally when the repository is changed, to refresh an already retrieved role use `ansible-galaxy install -f -r roles/requirements.yml`

A playbook to use this role would look like this:

```yaml
- hosts: master
  become: true
  gather_facts: true
  roles:
    - role: k3s-master
      when: install_k3s | bool
```

## Dependencies

None.

## Suggested project structure

```shell
├── inventories
│   ├── dev
│   │   ├── group_vars/
│   │   └── hosts.yml
│   └── prod
│       ├── group_vars/
│       └── hosts.yml
├── group_vars/
├── host_vars/
├── files/
├── templates/
├── roles
│   ├── local
│   │   ├── local_role1/
│   │   └── local_role2/
│   ├── requirements.yml
│   ├── .gitignore
├── ansible.cfg
├── README.md
├── some_playbook.yml
├── other_playbook.yml
```

Create a `roles/.gitignore` file to exclude the downloaded roles:

```ini
#Ignore everything in roles dir...
/*
# ... but current file...
!.gitignore
# ... external role requirement file
!requirements.yml
# ... and configured custom/local roles
!local*/
```

Add `roles_path = roles` to `ansible.cfg` to make sure that roles are searched and downloaded in our local folder.

### Workflow to deploy from the project

1. Clone the project repository
2. Download the external roles: `ansible-galaxy install -r roles/requirements`
3. Launch your playbook: `ansible-playbook -i inventories/dev some_playbook.yml -u ANSIBLE_USER`
