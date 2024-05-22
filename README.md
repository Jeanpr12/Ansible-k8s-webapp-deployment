# Ansible-k8s-webapp-deployment
Ansible Automation for Deploying Web Applications on AWS with Kubernetes


## Overview

This project demonstrates the automation of deploying a web application on AWS using Ansible and Kubernetes. It showcases essential skills for cloud infrastructure management, container orchestration, and automation.

## Project Structure

```
ansible-webapp-deployment
├── ansible.cfg
├── hosts
├── playbooks
│   ├── provision.yml
│   ├── setup_kubernetes.yml
│   ├── deploy_app.yml
│   └── site.yml
├── roles
│   ├── aws
│   │   ├── tasks
│   │   │   └── main.yml
│   ├── kubernetes
│   │   ├── tasks
│   │   │   └── main.yml
│   ├── webapp
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── deployment.yml.j2
│   │       ├── service.yml.j2
│   │       └── configmap.yml.j2
└── README.md
```

## Configuration Files

### `ansible.cfg`

```ini
[defaults]
inventory = hosts
remote_user = ubuntu
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
```
**Explanation:** Specifies inventory file, SSH user, private key for authentication, and disables host key checking.

### `hosts`

```ini
[aws]
ec2_instance ansible_host=<your-ec2-public-ip> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
```
**Explanation:** Inventory file defining the EC2 instance details and SSH authentication method.

## Playbooks

### `playbooks/provision.yml`

```yaml
---
- name: Provision AWS resources
  hosts: localhost
  tasks:
    - name: Launch EC2 instance
      ec2:
        key_name: your_key_pair
        instance_type: t2.micro
        image: ami-0c55b159cbfafe1f0
        wait: yes
        region: us-east-1
        group: default
        count: 1
        instance_tags:
          Name: WebAppServer
        vpc_subnet_id: subnet-xxxxxx
      register: ec2

    - name: Add new instance to host group
      add_host:
        hostname: "{{ item.public_ip }}"
        groupname: aws
      with_items: "{{ ec2.instances }}"
```
**Explanation:** Provisions an AWS EC2 instance and dynamically adds it to the Ansible hosts group.

### `playbooks/setup_kubernetes.yml`

```yaml
---
- name: Setup Kubernetes on AWS instances
  hosts: aws
  become: yes
  roles:
    - kubernetes
```
**Explanation:** Runs the Kubernetes setup role on the EC2 instance.

### `playbooks/deploy_app.yml`

```yaml
---
- name: Deploy web application using Kubernetes
  hosts: aws
  become: yes
  roles:
    - webapp
```
**Explanation:** Deploys the web application using Kubernetes manifests on the EC2 instance.

### `playbooks/site.yml`

```yaml
---
- import_playbook: provision.yml
- import_playbook: setup_kubernetes.yml
- import_playbook: deploy_app.yml
```
**Explanation:** Main playbook importing other playbooks for provisioning, Kubernetes setup, and application deployment.

## Roles

### `roles/aws/tasks/main.yml`

```yaml
---
# AWS-specific tasks (integrated in provision.yml)
```

### `roles/kubernetes/tasks/main.yml`

```yaml
---
- name: Install dependencies
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - software-properties-common
    state: present

- name: Download Google Cloud public signing key
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present

- name: Add Kubernetes apt repository
  apt_repository:
    repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
    state: present

- name: Update apt package index
  apt:
    update_cache: yes

- name: Install Kubernetes
  apt:
    name:
      - kubelet
      - kubeadm
      - kubectl
    state: present

- name: Initialize Kubernetes cluster
  command: kubeadm init --pod-network-cidr=10.244.0.0/16
  when: inventory_hostname == groups['aws'][0]

- name: Set kubeconfig for the ubuntu user
  command: >
    mkdir -p /home/ubuntu/.kube &&
    cp -i /etc/kubernetes/admin.conf /home/ubuntu/.kube/config &&
    chown ubuntu:ubuntu /home/ubuntu/.kube/config
  when: inventory_hostname == groups['aws'][0]

- name: Install Flannel CNI
  command: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  when: inventory_hostname == groups['aws'][0]
```
**Explanation:** Installs Kubernetes components, initializes the cluster, sets up `kubectl` for the user, and installs a CNI plugin (Flannel).

### `roles/webapp/tasks/main.yml`

```yaml
---
- name: Create Kubernetes deployment file
  template:
    src: deployment.yml.j2
    dest: /home/ubuntu/deployment.yml

- name: Create Kubernetes service file
  template:
    src: service.yml.j2
    dest: /home/ubuntu/service.yml

- name: Create Kubernetes configmap file
  template:
    src: configmap.yml.j2
    dest: /home/ubuntu/configmap.yml

- name: Deploy application
  command: kubectl apply -f /home/ubuntu/deployment.yml

- name: Deploy service
  command: kubectl apply -f /home/ubuntu/service.yml

- name: Deploy configmap
  command: kubectl apply -f /home/ubuntu/configmap.yml
```
**Explanation:** Templates Kubernetes manifests for deployment, service, and configmap, and applies them to the Kubernetes cluster.

### `roles/webapp/templates/deployment.yml.j2`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:latest
        ports:
        - containerPort: 80
      - name: db
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: root_password
```
**Explanation:** Defines the Kubernetes deployment for Nginx and MySQL containers.

### `roles/webapp/templates/service.yml.j2`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: webapp
```
**Explanation:** Defines a Kubernetes service to expose the web application.

### `roles/webapp/templates/configmap.yml.j2`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  root_password: root_password
```
**Explanation:** ConfigMap for storing MySQL root password.

## Usage

1. **Clone the repository:**
    ```bash
    git clone https://github.com/yourusername/ansible-webapp-deployment.git
    cd ansible-webapp-deployment
    ```

2. **Update the inventory file (`hosts`) with your EC2 instance details.**

3. **Run the main playbook:**
    ```bash
    ansible-playbook playbooks/site.yml
    ```

## Requirements

- Ansible installed on your local machine
- AWS account with configured CLI and necessary permissions
- SSH key pair for accessing EC2 instances



Feel free to customize the README further based on any additional details or preferences you might have. This file provides a comprehensive guide for users and potential recruiters to understand and replicate your project.
