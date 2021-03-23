# WordPress-Application on K8s-Cluster over AWS using Ansible !!
 *In this Blog you can understand how we can automate the Launching of Wordpress application on kubernetes-cluster using Ansible.*
 

<p align="center">
  <img width="460" height="300" src="https://s.w.org/about/images/logos/wordpress-logo-stacked-rgb.png">
</p>
 
**Wordpress** *is a free and open-source content management system written in PHP and paired with a **MySQL** or MariaDB database. Features include a plugin architecture and a template system, referred to within WordPress as Themes.

<p align="center">
  <img width="460" height="300" src="https://d1.awsstatic.com/asset-repository/products/amazon-rds/1024px-MySQL.ff87215b43fd7292af172e2a5d9b844217262571.png">
</p>

**MySQL** *is an open-source relational database management system. Its name is a combination of "My", the name of co-founder Michael Widenius's daughter, and "SQL", the abbreviation for Structured Query Language.

<p align="center">
  <img width="460" height="300" src="https://www.ovh.com/blog/wp-content/uploads/2019/01/kubernetesblog02.jpg">
</p>

**Kubernetes** *is an open-source container-orchestration system for automating computer application **deployment**, **scaling** and **management**. It was originally designed by Google and is now maintained by the Cloud Native Computing Foundation. It aims to provide a **platform for automating deployment, scaling, and operations of application containers across clusters of hosts**. It works with a range of container tools and runs containers in a cluster, often with images built using Docker. Kubernetes originally interfaced with the Docker runtime through a "Dockershim"; however, the shim has since been deprecated in favor of directly interfacing with containered or another CRI-compliant runtime.

<p align="center">
  <img width="460" height="300" src="https://i0.wp.com/volumes.blog/wp-content/uploads/2020/06/060120_1607_WhatisDellT1.png?w=760&ssl=1">
</p>

**Ansible** *is an automation tool that is used for **configuration management**. It is a very powerful tool written in python language, it has thousands of modules using which it works, Ansible gets its intelligence from its **modules**.

## Update the configuration file of Ansible

``` /etc/ansible/ansible.cfg ```

*EC2 instances allow key-based authentication, hence we must mention the path of the private key.

*The most important part is **Privilege Escalation**. Root powers are required if we want to configure anything in the instance. But ec2-user is a general user with limited powers. Privilege Escalation is used to give sudo powers to a general user.
```
[defaults]
inventory=location_of_inventory_file
command_warnings=false
remote_user = ec2-user
host_key_checking = false
private_key_file = /location_of_private_key
ask_pass = false

[privilege_escalation]
become = true
become_user = root
become_method = sudo
become_ask_pass = false
```

## How to create Ansible Role ?
**Roles** *let you automatically load related vars_files, tasks, handlers, and other Ansible artifacts based on a known file structure. Once you group your content in roles, you can easily reuse them and share them with other users.

*Below is the command you have to run for creating ansible-role.

``` ansible-galaxy init name_of_role ```

## Create Ansible-Role for Launching 3 AWS-instances

*Below is the content of tasks of Ansible-Role named **aws** for launching 1 master node and 1 Worker nodes. This role will launch the **AWS-instances** and dynamically allocate the IP's to the respective host groups of ansible.

``` 
- name: "Launching an AWS instance for Kubernetes Controller"
  ec2:
       key_name: "arpit_aws"
       instance_type: "t2.micro"
       image: "ami-08e0ca9924195beba"
       wait: yes
       count: 1
       instance_tags:
         Name: "K8s-Controller"
       vpc_subnet_id : "subnet-fc3f3694"
       assign_public_ip: yes
       state: present
       region: "ap-south-1"
       group_id: "sg-02f5ee0fc6b3ea0b6"
       aws_access_key: "{{ access_key }}"
       aws_secret_key: "{{ secret_key }}"
  register: control

- debug: 
       msg: "Successfully launched controller"  

- name: "Adding controller in a host group"
  add_host:
       hostname: "{{ item.public_ip }}"
       groups: controller
  loop: "{{ control['instances'] }}"

- name: "Waiting for SSH"
  wait_for:
       host: "{{ item.public_dns_name }}"
       port: 22
       state: started
  loop: "{{ control['instances'] }}"


- name: "Launching AWS instances for Kubernetes Worker Nodes"
  ec2:
       key_name: "arpit_aws"
       instance_type: "t2.micro"
       image: "ami-08e0ca9924195beba"
       wait: yes
       count: 1
       instance_tags:
              Name: "K8s-worker"
       vpc_subnet_id : "subnet-fc3f3694"
       assign_public_ip: yes
       state: present
       region: "ap-south-1"
       group_id: "sg-02f5ee0fc6b3ea0b6"
       aws_access_key: "{{ access_key }}"
       aws_secret_key: "{{ secret_key }}"
  register: worker

- debug:
       msg: "Successfully launched WorkerNodes"

- name: "Adding controller in a host group"
  add_host:
       hostname: "{{ item.public_ip }}"
       groups: controller
  loop: "{{ control['instances'] }}"

- name: "Waiting for SSH"
  wait_for:
       host: "{{ item.public_dns_name }}"
       port: 22
       state: started
  loop: "{{ control['instances'] }}"

- name: "Adding worker in a host group"
  add_host:
       hostname: "{{ item.public_ip }}"
       groups: workers
  loop: "{{ worker['instances'] }}"

- name: "Waiting for SSH"
  wait_for:
       host: "{{ item.public_dns_name }}"
       port: 22
       state: started
  loop: "{{ worker['instances'] }}"
```


→ ec2 is an ansible module that helps in launching the AWS-instances.

→ add_host is an ansible module that helps us to add IP dynamically in a temporary inventory variable. hostname holds the public IP of the instances.

→ wait_for is another ansible module that helps in checking whether the instances are ready. The public DNS of the instances can be used to check whether SSH service has started on port number 22 or not. Once the Instance is ready to do SSH the next module will be executed.

## Ansible tasks file of Controller Role for configuring Kubernetes-Master-Node

1. Master nodes and Worker nodes requires docker containers. The master node needs containers for running different services, while clients launches different pods on the worker node.

```
- name: "Installing Docker"  
  package:
        name: docker
        state: present
- name: "Enabling Docker"  
  service:
        name: docker
        state: started 
        enabled: yes
```

2. The next step is to install the software for Kubernetes setup which is kubeadm, but by default, kubeadm is not provided by the repos configured in the yum, so we need to configure yum first before downloading it.

``` 
- name: "Configuring yum for kubernetes"  
  yum_repository:        
          name: kubernetes
          description: kubernetes  
          baseurl:  https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
          enabled: yes
          gpgcheck: no
```

3. Install kubelet, kubectl, kubeadm, and iproute-tc.

```
- name: "Installing kubelet,kubectl,kubeadm and iproute-tc"
  package:
         name: "{{ item }}"     
         state: present
  loop: 
  - kubeadm
  - iproute-tc
```

4. Start the kubelet service.

```
- name: "Starting Kubelet Program" 
  service:
         name: kubelet     
         state: started  
         enabled: yes
```

5. All the services of the MasterNode are available as containers and “kubeadm” can be used to pull the required images.

```
- name: "Pulling images of kubeadm"  
  command: "kubeadm config images pull"
```

6. Docker by default uses a Cgroup Driver called cgroupfs. But Kubernetes doesn’t support this cgroup rather it supports the systemd driver.

``` 
- name: " Changing the cgroupdriver to systemd"  
  copy:
         dest: /etc/docker/daemon.json
         content: '{
                   "exec-opts": ["native.cgroupdriver=systemd"]
                   }'
```

7. Restart the Docker service

```
- name: "Restarting Docker"
  service:
          name: "docker"
          state: restarted
          enabled: yes
```

8. Configuring the network.
```
- name: "Configuring network"
  copy:
          dest: "/etc/sysctl.d/k8s.conf"
          content: "net.bridge.bridge-nf-call-ip6tables = 1\n
                    net.bridge.bridge-nf-call-iptables = 1"

- name: "Refreshing sysctl"
  shell: "sysctl --system"
```

9. Start the kubeadm service by providing all the required parameters.

```
- name: "Starting kubeadm service"
  command: "kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU --ignore-preflight-errors=Mem"
  ignore_errors: yes
```

10. We usually have a separate client who will use the kubectl command on the master, but just for testing, we can make the master as the client/user. Now if you run the “kubectl” command, it will fail (we already have kubectl software in the system). The reason for the above issue is that the client doesn’t know where the master is, so the client should know the port number of API, and username and password of the master, so to use this cluster as a normal user, we can copy some files in the HOME location, the files contain all the credentials of the master node.
```
- name: "Creating .kube Directory"
  file:
          path: "$HOME/.kube"
          state: directory
- name: "Copying config file"
  shell: "echo Y | cp -i /etc/kubernetes/admin.conf $HOME/.kube/config"
- shell: "chown $(id -u):$(id -g) $HOME/.kube/config
```

11. Install Add-ons.

```
- name: "Installing Addons"
  shell: "kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"
```  

## Ansible tasks file of Worker Role for configuring Kubernetes-Worker-Node.

```
- name: "Installing Docker"
  package:
        name: docker
        state: present   
   
- name: "Enabling Docker"
  service:
        name: docker  
        state: started
        enabled: yes

- name: "Configuring yum for kubernetes"
  yum_repository:
        name: kubernetes
        description: kubernetes
        baseurl: https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
        enabled: yes
        gpgcheck: no
        
- name: "Installing kubelet,kubectl,kubeadm and iproute-tc"
  package:
        name: "{{ item }}"
        state: present
  loop:
  - kubeadm
  - iproute-tc

- name: "Starting Kubelet Program"
  service:
         name: kubelet
         state: started
         enabled: yes

- name: " Changing the cgroupdriver to systemd"
  copy:
         dest: /etc/docker/daemon.json
         content: '{
                   "exec-opts": ["native.cgroupdriver=systemd"]
                   }'
- name: "Restarting Docker"
  service:
          name: "docker"
          state: restarted
          enabled: yes

- name: "Configuring network"
  copy:
          dest: "/etc/sysctl.d/k8s.conf"
          content: "net.bridge.bridge-nf-call-ip6tables = 1\n
                    net.bridge.bridge-nf-call-iptables = 1"

- name: "Refreshing sysctl"
  shell: "sysctl --system"


- debug: 
       msg: "Worker nodes has been successfully Configured"
```

## Now, Integrating all the Ansible-roles in a single playbook to configure the cluster and also mentioning the tasks for setting up WordPress and MySQL.

```
- hosts: localhost
  roles:
  - role: aws

- hosts: controller
  roles:
  - role: controller

- hosts: workers
  roles:
  - role: worker     

- hosts: controller
  tasks:
  - shell:  "kubectl create deployment mywp --image=wordpress:5.1.1-php7.3-apache"   

  - shell: "kubectl run mydb --image=mysql:5.7  --env=MYSQL_USER=arpit  --env=MYSQL_PASSWORD=redhat --env=MYSQL_DATABASE=wpdb  --env=MYSQL_ROOT_PASSWORD=redhat"
    
  - shell: "kubectl expose deployment mywp --port 80 --type=NodePort"
  
  - name: "get service"
    shell: "kubectl get svc"
    register: service
  
  - debug:
      var: "service.stdout_lines" 
  
  - name: "Pausing playbook for 50 seconds"
    pause: 
      seconds: 50
  
  - name: "gettin database IP"
    shell: "kubectl get pods -o wide"
    register: Database
  
  - debug:
      var: "Database.stdout_lines"
 ```
 
# Your Kubernetes-Cluster will be successfully configured on top of AWS !

Thanks for reading this blog !
