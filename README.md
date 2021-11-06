# Installing and configuring Wordpress using Ansible

The infrastructure here includes one ansible master server and a ansible client server. Ansible is installed in the Master server and using ansible playbook we will be installing and configuring Apache, MariaDB and Wordpress in the client server.

# Ansible
Ansible is a radically simple IT automation engine that automates cloud provisioning, configuration management, application deployment, intra-service orchestration, and many other IT needs. It is the simplest way to automate IT. Ansible is the only automation language that can be used across entire IT teams from systems and network administrators to developers and managers.
You can learn more about ansible : 
- https://www.ansible.com
- https://www.ansible.com/overview/how-ansible-works

# Intro to playbooks
Ansible Playbooks offer a repeatable, re-usable, simple configuration management and multi-machine deployment system, one that is well suited to deploying complex applications. If you need to execute a task with Ansible more than once, write a playbook and put it under source control. Then you can use the playbook to push out new configuration or confirm the configuration of remote systems.
You can learn more about the same in the [website.](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html)

# Installing Ansible
> Here I am using AWS Amazon linux server
```sh
sudo amazon-linux-extras install ansible2 -y
```
Please see the [website](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installation-guide) for more information.

# 1. Creating an inventory file
> Here, I am creating a inventory file in the name of "hosts". The name of the file can be anything and this file contains the details of the remote servers(client) that you want Ansible to run against.
You can learn more about the same in the [website.](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)

```sh
vim hosts
```
```sh
[amazon]
IP   ansible_user="ec2-user" ansible_port="22" ansible_private_key_file="ansible.pem"
```

# 2. Creating variable files
- Here, I am defining some variables that will be used ro run in the ansible-playbook.
- Extension of the files will be .vars

You can learn more about the same in the [website.](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html)

- Defining some variables for apache.
```sh
vim apache.vars
```
```sh
---
httpd_port: "80"  
httpd_user: "apache"
httpd_group: "apache"
```
- Defining a variable for a domain name

```sh
vim domain.vars
````
```sh
---
httpd_domain: "www.example.com"
```
 
- Defining some variables for Mariadb
```sh
vim mariadb.vars
```
```
---
sql_root_password: "root@12345"
sql_extra_user: "wordpress"
sql_extra_user_password: "wordpress@123"
sql_extra_database: "wordpress"
```

- Defining some variables for wordpress
```sh
vim wordpress.vars
```
```sh
---
wordpress_url: "https://wordpress.org/wordpress-5.7.3.tar.gz"
```

# 3. Apache configuration file
- Making some changes in apache configuration file

```sh
vim httpd.conf.tmpl
```
Adding the following values in the httpd.conf file and saving the file as httpd.conf.tmpl
```sh
#Listen 12.34.56.78:80
Listen {{ httpd_port }}
User {{ httpd_user }}
Group {{ httpd_group }}
```

# 4. Adding a vhost entry
```sh
vim vhost.conf.tmpl
```
```sh
<virtualhost *:{{ httpd_port }}>
  
  servername {{ httpd_domain }}
  documentroot /var/www/html/{{ httpd_domain }}
  directoryindex index.html index.php
  <directory /var/www/html/{{ httpd_domain }}>
     allowoverride all
  </directory>
    
</virtualhost>
```

# 5. Making updates to the Wordpress wp-config.php file
> Updating the database, username and password details in the file and saving it as wp-config.php.tmpl

```sh
vim wp-config.php.tmpl
```
Adding the values to the file
```sh
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', '{{ sql_extra_database }}' );

/** MySQL database username */
define( 'DB_USER', '{{ sql_extra_user }}' );

/** MySQL database password */
define( 'DB_PASSWORD', '{{ sql_extra_user_password }}' );
```

# 6. Creating Playbook
```sh
---
- name: "Wordpress Installation"
  hosts: amazon
  become: true
  vars_files:
    - apache.vars
    - domain.vars
    - wordpress.vars
    
  vars_prompt:
    - name: sql_root_password
      prompt: Enter the root password for MySQL
      private: yes

    - name: sql_extra_user
      prompt: Enter the Database User Name for your Wordpress

    - name: sql_extra_user_password
      prompt: Enter the Password for the Database user for your Wordpress
      private: yes

    - name: sql_extra_database
      prompt: Enter the database name for your Wordpress  
    
  tasks:
```
- ## Task  - Apache
```sh
################################## Apache ##############################################

    - name: "Apache Installing PHP"
      shell: amazon-linux-extras install php7.4 -y
      tags:
        - lamp

    - name: "Apache - Installing httpd"
      yum:
        name: httpd
        state: present
      tags:
        - lamp

    - name: "Apache - copying httpd.conf"
      template:
        src: httpd.conf.tmpl
        dest: /etc/httpd/conf/httpd.conf
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
      tags:
        - lamp

    - name: "Apache - copying vhost.conf.tmpl"
      template:
        src: vhost.conf.tmpl
        dest:  "/etc/httpd/conf.d/{{ httpd_domain }}.conf"
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
      tags:
        - lamp


    - name: "Apache - Creating DocumentRoot"
      file:
        path: "/var/www/html/{{ httpd_domain }}"
        state: directory
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}" 
      tags:
        - lamp

    - name: "Apache - Restarting/Enabling httpd Service"
      service:
        name: httpd
        state: restarted
        enabled: true
      tags:
        - lamp

```
