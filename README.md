# WordPress-Application on K8s-Cluster over AWS using Ansible !!
 In this Blog you can understand how we can automate the Launching of Wordpress application on kubernetes-cluster using Ansible.
 
![Wordpress](https://s.w.org/about/images/logos/wordpress-logo-stacked-rgb.png)
 is a free and open-source content management system written in PHP and paired with a MySQL or MariaDB database. Features include a plugin architecture and a template system, referred to within WordPress as Themes.

![MySQL](https://d1.awsstatic.com/asset-repository/products/amazon-rds/1024px-MySQL.ff87215b43fd7292af172e2a5d9b844217262571.png)
MySQL is an open-source relational database management system. Its name is a combination of "My", the name of co-founder Michael Widenius's daughter, and "SQL", the abbreviation for Structured Query Language.

![k8s](https://www.ovh.com/blog/wp-content/uploads/2019/01/kubernetesblog02.jpg)

Kubernetes is an open-source container-orchestration system for automating computer application deployment, scaling, and management. It was originally designed by Google and is now maintained by the Cloud Native Computing Foundation. It aims to provide a "platform for automating deployment, scaling, and operations of application containers across clusters of hosts". It works with a range of container tools and runs containers in a cluster, often with images built using Docker. Kubernetes originally interfaced with the Docker runtime through a "Dockershim"; however, the shim has since been deprecated in favor of directly interfacing with containered or another CRI-compliant runtime.

![Ansible](https://i0.wp.com/volumes.blog/wp-content/uploads/2020/06/060120_1607_WhatisDellT1.png?w=760&ssl=1)

## How to create Ansible Role ?
Roles let you automatically load related vars_files, tasks, handlers, and other Ansible artifacts based on a known file structure. Once you group your content in roles, you can easily reuse them and share them with other users.
Below is the command you have to run for creating ansible-role.
```ansible-galaxy init name_of_role
