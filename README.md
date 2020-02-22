# Provide a highly available Jenkins cluster using Kubernetes and Ansible 

I want to create a Kubernetes cluster network on which I would be installing installing Jenkins. 
For creating Kubernetes Cluster, I would be using Ansible machines on CentOS 7 from the Google Cloud Platform.

I can directly bring up Kubernetes clusters using GKE or EKS, 
But, for initial runs with economy in mind, I need the infrastructure for the clusters on my local servers. 
I would hence use Ansible to create cluster network. I want to go first with the hard way!

## Installation of Ansible Master

CentOS 7 machines for this project.

### 1. Install epel-release

```bash
sudo yum install epel-release -y
```
### 2. Update packages 

```bash
sudo yum update -y
```
### 3. Install Ansible 

```bash
sudo yum install ansible -y
```
### 4. Switch to root user

```bash
 sudo su
```
### 5. Create an ansible user and provide password

```bash
useradd ansible
passwd ansible
```
 Note: I set password as '********' to meet the requirements.

### 6. Give permissions to ansible user

```bash
visudo
```
#### Go to the end and you will see some root permission privilege. 
Below " root  ALL=(ALL) ALL ", give permissions to ansible user as follows:
```bash
ansible ALL=(ALL)  NOPASSWD:ALL
```
### 7. Setup Password Authentication

```bash
cd ../../../
vi /etc/sshd/sshd_config
```
 Set => passwordauthentication yes
#### DEFAULT => passwordauthentication no

#### Now, restart the service using the command
```bash
service sshd restart
```
#### Now, I have an ansible machine with ansible user given sudo privileges.

### 8. Switch to ansible user and generate ssh-key 
```bash
su ansible
ssh-keygen
```

### 9. Establish connection with the localhost
```bash
ssh-copy-id localhost
```
SSH into the machine using "ssh localhost" to confirm access for ansible user without password.

## Installation for Ansible Nodes

### 1. Follow steps 1 to 7 above except installing ansible on nodes.

### 2. Switch back to Ansible master and copy ssh key to the node

```bash
ssh-copy-id <server id>
```

#### Note: To find the hostname of the node, use 
```bash
hostname -f
```  
#### Now, we have a working master and node with Ansible up and running.

## Creating Kubernetes Cluster on Master and Nodes
### 1. Creating hosts file
```bash 
vi hosts
```
Paste the commands:
```bash
[masters]
master ansible_host=ip_address ansible_user=ansible

[workers]
node1 ansible_host=ip_address ansible_user=ansible
node2 ansible_host=ip_address ansible_user=ansible
node3 ansible_host=ip_address ansible_user=ansible
```
### 2. Create a directory named kube_cluster_playbooks
```bash
mkdir kube_cluster_playbooks
cd kube_cluster_playbooks
```
We will be writing all the playbooks inside this directory

### 3. Setting up dependencies for K8s cluster installation
```bash
vi kube-dependencies.yml
```
Copy this instruction:
```bash 

```
- hosts: all  
  become: true
  tasks:
  
  ## Install Docker
   - name: install Docker 
     yum:
       name: docker
       state: present
       update_cache: true

  ## Start Docker Service
   - name: start Docker 
     service:
       name: docker
       state: started

  ## Disable SELinux : Else will cause issues due to not being fully supported by kubernetes Cluster
   - name: disable SELinux 
     command: setenforce 0

   - name: disable SELinux on reboot
     selinux:
       state: disabled
  
  ## Set up IPTables for Kubernetes Networking
   - name: ensure net.bridge.bridge-nf-call-ip6-tables set to 1 
     sysctl:
      name: net.bridge.bridge-nf-call-ip6tables
      value: 1
      state: present

   - name: ensure net.bridge.bridge-nf-call-ip-tables set to 1
     sysctl:
      name: net.bridge.bridge-nf-call-iptables
      value: 1
      state: present

  ## Add Kubernetes YUM repository in remote server' repository list
   - name: add Kubernetes' YUM repository 
     yum_repository:
      name: Kubernetes
      description: Kubernetes YUM repository
      baseurl: https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
      gpgkey: https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
      gpgcheck: yes

  ## Install Kubelet
   - name: install kubelet 
     yum:
        name: kubelet-1.14.0
        state: present
        update_cache: true

  ## Install Kubeadm
   - name: install kubeadm 
     yum:
        name: kubeadm-1.14.0
        state: present

  ## Start Kubelet Service
   - name: start kubelet 
     service:
       name: kubelet
       enabled: yes
       state: started

  ## Install Kubectl on Master Node
- hosts: master
  become: true
  tasks:
   - name: install kubectl 
     yum:
        name: kubectl-1.14.0
        state: present
        allow_downgrade: yes

### 4. Master installation
```bash
- hosts: master
  become: true
  tasks:

  ## Initialize Kubernetes cluster on master   
    - name: initialize the cluster 
      shell: kubeadm init --pod-network-cidr=10.244.0.0/16 >> cluster_initialized.txt
      args:
        chdir: $HOME
        creates: cluster_initialized.txt

  ## Create /.kube directory for storing configuration data
    - name: create .kube directory 
      become: true
      become_user: ansible
      file:
        path: $HOME/.kube
        state: directory
        mode: 0755

  ## Copy files created by "kubeadm init" to ansible user directory 
    - name: copy admin.conf to users kube config 
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/ansible/.kube/config
        remote_src: yes
        owner: ansible

  ## Install flannel network as Pod Networking plugin for Pod communication
    - name: install Pod network 
      become: true
      become_user: ansible
      shell: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml >> pod_network_setup.txt
      args:
        chdir: $HOME
        creates: pod_network_setup.txt
  ```
  ### 5. Worker installation
```bash
- hosts: master
  become: true
  gather_facts: false
  tasks:
  ## Join k8s worker nodes to the k8s master node by creating tokens in master and then setting join command
    - name: get join command 
      shell: kubeadm token create --print-join-command
      register: join_command_raw

    - name: set join command
      set_fact:
        join_command: "{{ join_command_raw.stdout_lines[0] }}"

  ## Join nodes to master
- hosts: workers
  become: yes
  tasks:
    - name: join cluster 
      shell: "{{ hostvars['master'].join_command }} --ignore-preflight-errors all  >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt
```
### 7. Create playbook k8s_cluster which executes all playbooks kube-dependencies.yml, master.yml, workers.yml in order

```bash 
cd ..
vi k8s-cluster.yml
```
Paste these commands:
```bash
---
- import_playbook: kube_cluster_playbooks/kube-dependencies.yml
- import_playbook: kube_cluster_playbooks/master.yml
- import_playbook: kube_cluster_playbooks/workers.yml 
```
### 8. Execute the playbook 
```bash
ansible-playbook k8s_cluster.yml -i hosts
```
### 9. Check the clusters
```bash
kubectl get nodes
```
The output shows active Kubernetes master and 3 nodes in a cluster READY 

```bash 
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   3m   v1.14.0
node1    Ready    <none>   3m   v1.14.0
node2    Ready    <none>   3m   v1.14.0
node3    Ready    <none>   3m   v1.14.0
```
On this Kubernetes cluster, I will install Jenkins with 1 master and 2 nodes on top of Kubernetes and let Kubernetes orchestrate the Jenkins server

## Jenkins Installation on Kubernetes
### 1. Create jenkins-deployment.yml
```bash
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: master
    spec:
      containers:
      - name: master
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        - containerPort: 50000
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 2
          failureThreshold: 5
        env:
        - name: JAVA_OPTS
          value: "-Djenkins.install.runSetupWizard=false"
```
### 2. Create jenkins-service.yml
```bash
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  externalTrafficPolicy: Cluster
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: master
  type: LoadBalancer
```
### 3. Execute Jenkins cluster
```bash 
kubectl apply -f jenkins-deployment.yml
kubectl apply -f jenkins-service.yml
```
Check the status using:
```bash 
kubectl get pod -o wide
```
You should be able to see
```bash
$ kubectl get pod -o wide
NAME                                 READY     STATUS    RESTARTS   AGE
jenkins-72205450-mc851               1/1       Running   0          25s
```
### 4. Port forward to access Jenkins UI
```bash
kubectl port-forward jenkins-72205450-mc851 8080
```

You should be seeing
```bash
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Access it using https://ip_address_of_pod:8080

Terminate the session using Ctrl + C


        





## Contributing
Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.


