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
vim main.yml
```
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
## Tasks  - Apache
```sh
################################## Apache ##############################################
    - name: "Installing PHP"
      shell: amazon-linux-extras install php7.4 -y
      tags:
        - lamp

    - name: "Installing httpd"
      yum:
        name: httpd
        state: present
      tags:
        - lamp

    - name: "copying httpd.conf"
      template:
        src: httpd.conf.tmpl
        dest: /etc/httpd/conf/httpd.conf
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
      tags:
        - lamp

    - name: "copying vhost.conf.tmpl"
      template:
        src: vhost.conf.tmpl
        dest:  "/etc/httpd/conf.d/{{ httpd_domain }}.conf"
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
      tags:
        - lamp

    - name: "Creating DocumentRoot"
      file:
        path: "/var/www/html/{{ httpd_domain }}"
        state: directory
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}" 
      tags:
        - lamp

    - name: "Restarting httpd Service"
      service:
        name: httpd
        state: restarted
        enabled: true
      tags:
        - lamp
```
## Tasks  - Mariadb
```sh
####################### Mariadb ##################################################
    - name: "Mariadb Installation"
      yum:
        name: 
          - mariadb-server
          - MySQL-python
        state: present

    - name: "Mariadb - Restart"
      service:
        name: mariadb
        state: restarted
        enabled: true
      tags:
        - lamp
        - mariadb

    - name: "Mariadb-Reset Root Password"
      ignore_errors: true
      mysql_user:
        login_user: "root"
        login_password: ""
        user: "root"
        password: "{{ sql_root_password }}"
        host_all: true
      tags:
        - lamp
        - mariadb 
        
    - name: "Removing Anonymous Users"
      mysql_user:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        user: ""
        state: absent
      tags:
        - lamp
        - mariadb  

    - name: "Removing test database"
      mysql_db:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        name: "test"
        state: absent
      tags:
        - lamp
        - mariadb

    - name: "Creating extra database for wordpress"
      mysql_db:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        name: "{{ sql_extra_database }}"
        state: present
      tags:
        - lamp
        -  mariadb
        
    - name: "Creating Extra user for wordpress"
      mysql_user:
        login_user: "root"
        login_password: "{{ sql_root_password }}"
        user: "{{ sql_extra_user }}"
        password: "{{ sql_extra_user_password }}"
        state: present
        priv: '{{ sql_extra_database }}.*:ALL'
      tags:
        - lamp
        - mariadb
```
## Tasks -  Wordpress
```sh
######################### Wordpress ######################################################
    - name: "Downloading wordpress"
      get_url:
        url: "{{ wordpress_url }}"
        dest:  "/tmp/wordpress.tar.gz"  

    - name: "Extracting Archive File"
      unarchive:
        src: "/tmp/wordpress.tar.gz"
        dest: "/tmp/"
        remote_src: true

    - name: "Copying Contents"
      copy:
        src: "/tmp/wordpress/"
        dest: "/var/www/html/{{ httpd_domain }}/"
        remote_src: true
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"      

    - name: "Creating wp-config.php"
      template:
        src: wp-config.php.tmpl
        dest: "/var/www/html/{{ httpd_domain }}/wp-config.php"
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
```
## Tasks -  Post Installation
```sh
###################### Post Installation ################################################
    - name: "Post-Installation - Clean Up"
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - "/tmp/wordpress"
        - "/tmp/wordpress.tar.gz"

    - name: "Post-Installation - Restart services"
      service:
        name: "{{ item }}"
        state: restarted
        enabled: true
      with_items:
        - httpd
        - mariadb      
```
## Execution
> Make sure the Playbook, inventory file and the SSH key to access client server in working directory of the Ansible Master server.

- Running a syntax check
```sh
ansible-playbook -i hosts main.yml --syntax-check
```
- Executing the Playbook
```sh
ansible-playbook -i hosts main.yml
```
