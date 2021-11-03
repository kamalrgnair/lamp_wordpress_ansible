# Installing WordPress on LAMP stack using Ansible Playbook

Installation and setting up the WordPress website manually require many steps, lot of things we need to perform before setting up a WordPress website like installing and configuring Apache, installing PHP, Database server.  These process or task can be automated by writing Ansible playbook.

Ansible is an open source IT automation engine that automates provisioning, configuration management, application deployment, orchestration, and many other IT processes. Ansible is agentless, which means no additional softawares needs to be installed on client servers for managing the process, used  SSH protocol to connect to servers and run tasks. Ansible contains modules using which we can automate task, if builtin modules are not used then ad-hoc commands will help to accomplish tasks

#### Prerequisites

---

Install Ansible and verify the connection between Ansible server and client machine on which you need to automate tasks. Ansible can be installed using PIP or package manager.  In [here](https://docs.ansible.com/ansible/latest/installation_guide/index.html) you can have more details on Ansible installation.

In this project I'am using an EC2 instance for both Ansible server and client/target. After Ansible installation for connection from Ansible server to client, we need to create an inventory file to define our target.

```
[root@kamal-workbox lamp_wordpress_project]# pwd
/root/lamp_wordpress_project

[root@kamal-workbox lamp_wordpress_project]# vim hosts    //hosts - inventory file

[root@kamal-workbox lamp_wordpress_project]# cat hosts
[lamp_wordpress]
52.66.138.8 ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="mykey.pem"
[root@kamal-workbox lamp_wordpress_project]# 
```

*Note: You may use ssh-key or password for authenticating.*

**Ping the target**

---

```
[root@kamal-workbox lamp_wordpress_project]# ansible -i hosts lamp_wordpress -m ping
[WARNING]: Platform linux on host 65.2.34.84 is using the discovered Python interpreter at
/usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more
information.
52.66.138.8 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
[root@kamal-workbox lamp_wordpress_project]#
```

You will get output like above if the ping is successfull. You can use ```all``` in place of particluar target for pinging all targets.

**Create roles for the tasks**

---

Roles let you automatically load related vars, files, tasks, handlers, and other Ansible artifacts based on a known file structure. In this project I'm using two roles ```lamp``` and ```wordpress``` for automating the task. Roles can be created using   ```init``` command.

```
[root@kamal-workbox roles]# ansible-galaxy init lamp
- Role lamp was created successfully
[root@kamal-workbox roles]# ansible-galaxy init wordpress
- Role wordpress was created successfully
[root@kamal-workbox roles]#
```

Directory listing for both the roles will be like

```
[root@kamal-workbox roles]# tree lamp
lamp
|-- defaults
|   `-- main.yml
|-- files
|-- handlers
    |   `-- main.yml
|-- meta
|   `-- main.yml
|-- README.md
|-- tasks
|   `-- main.yml
|-- templates
|-- tests
|   |-- inventory
|   `-- test.yml
`-- vars
    `-- main.yml

8 directories, 8 files

```

```
[root@kamal-workbox roles]# tree wordpress
wordpress
|-- defaults
|   `-- main.yml
|-- files
|-- handlers
|   `-- main.yml
|-- meta
|   `-- main.yml
|-- README.md
|-- tasks
|   `-- main.yml
|-- templates
|-- tests
|   |-- inventory
|   `-- test.yml
`-- vars
    `-- main.yml

8 directories, 8 files
[root@kamal-workbox roles]
```

* **lamp role**

In lamp role I'm going to write codes for 

​                                                   *  *Install Apache webserver*

​                                                   *  *Install PHP* 

​                                                   *  *Install MariaDB database server*

​                                                   * *Creating a user account and setting up document root for domain*

​                                                   * *Creating domain virtual host*

These main tasks for lamp role are written to file ```    lamp/tasks/main.yml```                               

```
[root@kamal-workbox tasks]# pwd
/root/lamp_wordpress_project/roles/lamp/tasks

[root@kamal-workbox tasks]# cat main.yml
###############ApacheWebserver###############          

- name: "Installing Apache-Webserver"
  yum:
    name: httpd
    state: present
  notify:
    - restart_httpd_service
  
- name: "Starting Apache-Webserver-service"
  service:
    name: httpd
    state: started

- name: "Enabling Apache-Webserver-service"
  service:
    name: httpd
    enabled: true

###############Mariadbserver#################        

- name: "Installing Mariadb server and dependency"
  yum:
    name: 
      - mariadb-server
      - MySQL-python            
    state: present
      
- name: "Starting Mariadb server"
  service:
    name: mariadb
    state: started        

- name: "Enabling Maridb server"
  service:
    name: mariadb
    enabled: true

- name: "Setting Mariadb root password"
  ignore_errors: true
  mysql_user:
    login_user: "root"
    login_password: ""
    user: "root"
    password: "{{ mysql_root_password }}"
    host_all: true
      
- name: "Removing anonymous users"
  mysql_user:
    login_user: "root"
    login_password: "{{ mysql_root_password }}"
    name: ""
    state: absent
    host_all: true

- name: "Removing Test databases"
  mysql_db:
    login_user: "root"
    login_password: "{{ mysql_root_password }}"
    name: "test"
    state: absent

- name: "setting MySql root login without prompting"
  template:
    src: cnf.tmpl
    dest: "/root/.my.cnf"
    owner: "root"
    group: "root"

###############PHP Installation##############        
- name: "Installing PHP"
  shell: amazon-linux-extras install php8.0 -y
  notify:
    - restart_httpd_service    
###############Setting Docroot for Domain####

- name: "Add user account"
  user:  
    name: "{{ username }}"
    comment: "{{ username}}"
    shell: "{{ shell_access}}"
    password: "{{ user_password | password_hash('sha512') }}"

- name: "Set secure persmission for {{ username }} home directory"
  file: 
    path: "/home/{{ username }}"
    state: directory
    mode: 0711

- name: "Creating public_html directory"
  file:
    path: "/home/{{ username }}/public_html"
    state: directory
    owner: "{{ username }}"
    group: "{{ username }}"        

###############Creating domain virtualhost###

- name: "Creating {{ domain_name }} virtualhost"
  template:
    src: virtualhost.tmpl
    dest: "/etc/httpd/conf.d/{{ domain_name }}.conf"
    owner: "root"
    group: "root"
  notify:
    - restart_httpd_service
[root@kamal-workbox tasks]#
```

*Variables for lamp role*

Variables for this role is declared in file ```lamp/vars/main.yml```

```
[root@kamal-workbox vars]# pwd
/root/lamp_wordpress_project/roles/lamp/vars

[root@kamal-workbox vars]# cat main.yml
###############Lamp variables################
mysql_root_password: "mypassword#123"
username: "mkduo"
user_password: new123@456
shell_access: "/bin/bash"
web_port: 80
domain_name: "mkduo.in"
[root@kamal-workbox vars]#
```

*Adding handlers to playbook*

Handlers are tasks that only run when notified. Handlers are used in this project to restart apache webserver whenever configuration of this service got changed by task, ```notify```directive is used for this. In the ```lamp/tasks/main.yml``` file you can see ```notify``` directive is used when ever those taks modify this service configuration. Regardless of how many tasks notify a handler, it will run only once, after all of the tasks completed in a playbook.

Handlers tasks are written to ```lamp/handlers/main.yml``` file

```
[root@kamal-workbox handlers]# pwd
/root/lamp_wordpress_project/roles/lamp/handlers

[root@kamal-workbox handlers]# cat main.yml 
---
- name: "restart_httpd_service"
  service:
    name: httpd
    state: restarted
[root@kamal-workbox handlers]#
```

*Adding Templates*

Create two configuration files  "cnf.tmpl" and "virtualhost.tmpl" in ```lamp/templates``` directory for mysql root login with prompting password and other for domain virtualhost, these files will be placed in client server on execution of playbook via template module. 

```
[root@kamal-workbox templates]# pwd
/root/lamp_wordpress_project/roles/lamp/templates

[root@kamal-workbox templates]# cat cnf.tmpl
[client]
user=root
password="{{ mysql_root_password }}"
[root@kamal-workbox templates]# 
```

```
[root@kamal-workbox templates]# pwd
/root/lamp_wordpress_project/roles/lamp/templates

[root@kamal-workbox templates]# cat virtualhost.tmpl
<VirtualHost *:{{web_port}}>
  ServerName {{ domain_name }}
  ServerAlias {{ domain_name }} www.{{ domain_name }}
  DocumentRoot /home/{{username}}/public_html
  <Directory /home/{{username}}/public_html>
      Require all granted
      AllowOverride all
  </Directory>
</VirtualHost>
[root@kamal-workbox templates]#
```



* **wordpress role**

Worpress role is for installing latest WorPress, setting worpdress database, user and privilage.

Main playbook for this role is written to ```wordpress/tasks/main.yml```

```
[root@kamal-workbox tasks]# pwd
/root/lamp_wordpress_project/roles/wordpress/tasks

[root@kamal-workbox tasks]# cat main.yml
###############Setting DB,user and privilege#
    - name: "Create WordPress DB"
      mysql_db:
        name: "{{ wp_db }}"
        state: present          

    - name: "Create Wordpress database user"
      mysql_user:
        user: "{{ wp_db_user }}"
        password: "{{ wp_user_pass }}"
        state: present
        priv: "{{ wp_db }}.*:ALL"

    - name: "Setting wordpress config"
      template: 
        src: wp_config.tmpl
        dest: "/home/{{ username }}/public_html/wp-config.php"
        owner: "{{ username }}"
        group: "{{ username }}"

    - name: "Removing Wordpress Archive and extracted data"
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - "/home/{{ username }}/latest.zip"
        - "/home/{{ username }}/wordpress"
```

*Variables for wordpress role*

```
[root@kamal-workbox vars]# pwd
/root/lamp_wordpress_project/roles/wordpress/vars

[root@kamal-workbox vars]# cat main.yml 
wp_db: "{{ username }}_wpdb"
wp_db_user: "{{username }}_wpusr"
wp_user_pass: "wp@mkduo"
wordpress_latest: "https://wordpress.org/latest.zip"
[root@kamal-workbox vars]#
```

*Adding Templates*

Adding WordPress configuration template with variables defined in a file under ```wordpress/templates``` 

```
[root@kamal-workbox templates]# pwd
/root/lamp_wordpress_project/roles/wordpress/templates

[root@kamal-workbox templates]# cat wp_config.tmpl
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the web site, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * MySQL settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://wordpress.org/support/article/editing-wp-config-php/
 *
 * @package WordPress
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', '{{ wp_db }}' );

/** MySQL database username */
define( 'DB_USER', '{{ wp_db_user }}' );

/** MySQL database password */
define( 'DB_PASSWORD', '{{ wp_user_pass }}' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication unique keys and salts.
 *
 * Change these to different unique phrases! You can generate these using
 * the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}.
 *
 * You can change these at any point in time to invalidate all existing cookies.
 * This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );

/**#@-*/

/**
 * WordPress database table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://wordpress.org/support/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', false );

/* Add any custom values between this line and the "stop editing" line. */



/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';
[root@kamal-workbox templates]#
```

#### Writing main playbook file for this project

Main playbook file for executing this project is under ```/root/lamp_wordpress_project``` diretcory.

```
[root@kamal-workbox lamp_wordpress_project]# pwd
/root/lamp_wordpress_project

[root@kamal-workbox lamp_wordpress_project]# cat lamp_wordpress.yml 
---
- name: "Lamp-wordpress"
  hosts: lamp_wordpress
  become: true
  roles:
    - lamp
    - wordpress      
[root@kamal-workbox lamp_wordpress_project]# 
```

Check playbook syntax and run the Playbook 

```
[root@kamal-workbox lamp_wordpress_project]# ansible-playbook -i hosts lamp_wordpress.yml --syntax-check

playbook: lamp_wordpress.yml
```

```
[root@kamal-workbox lamp_wordpress_project]# ansible-playbook -i hosts lamp_wordpress.yml

PLAY [Lamp-wordpress] ****************************************************************************

TASK [Gathering Facts] ***************************************************************************
[WARNING]: Platform linux on host 52.66.138.8 is using the discovered Python interpreter at
/usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more
information.
ok: [52.66.138.8]

TASK [lamp : Installing Apache-Webserver] ********************************************************
changed: [52.66.138.8]

TASK [lamp : Starting Apache-Webserver-service] **************************************************
changed: [52.66.138.8]

TASK [lamp : Enabling Apache-Webserver-service] **************************************************
changed: [52.66.138.8]

TASK [lamp : Installing Mariadb server and dependency] *******************************************
changed: [52.66.138.8]

TASK [lamp : Starting Mariadb server] ************************************************************
changed: [52.66.138.8]

TASK [lamp : Enabling Maridb server] *************************************************************
changed: [52.66.138.8]

TASK [lamp : Setting Mariadb root password] ******************************************************
[WARNING]: Module did not set no_log for update_password
changed: [52.66.138.8]

TASK [lamp : Removing anonymous users] ***********************************************************
changed: [52.66.138.8]

TASK [lamp : Removing Test databases] ************************************************************
changed: [52.66.138.8]

TASK [lamp : setting MySql root login without prompting] *****************************************
changed: [52.66.138.8]

TASK [lamp : Installing PHP] *********************************************************************
changed: [52.66.138.8]

TASK [lamp : Add user account] *******************************************************************
changed: [52.66.138.8]

TASK [lamp : Set secure persmission for mkduo home directory] ************************************
changed: [52.66.138.8]

TASK [lamp : Creating public_html directory] *****************************************************
changed: [52.66.138.8]

TASK [lamp : Creating mkduo.in virtualhost] ******************************************************
changed: [52.66.138.8]

TASK [Downloading latest wordpress] **************************************************************
changed: [52.66.138.8]

TASK [wordpress : Extracting WordPress archive] **************************************************
changed: [52.66.138.8]

TASK [wordpress : Copying WordPress files to public_html] ****************************************
changed: [52.66.138.8]

TASK [wordpress : Create WordPress DB] ***********************************************************
changed: [52.66.138.8]

TASK [wordpress : Create Wordpress database user] ************************************************
changed: [52.66.138.8]

TASK [Setting wordpress config] ******************************************************************
changed: [52.66.138.8]

TASK [wordpress : Removing Wordpress Archive and extracted data] *********************************
changed: [52.66.138.8] => (item=/home/mkduo/latest.zip)
changed: [52.66.138.8] => (item=/home/mkduo/wordpress)

RUNNING HANDLER [lamp : restart_httpd_service] ***************************************************
changed: [52.66.138.8]

PLAY RECAP ***************************************************************************************
52.66.138.8                : ok=24   changed=23   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[root@kamal-workbox lamp_wordpress_project]# 
```

#### Conclusion

Using the steps in this document I've setup lamp and deploy wordpress onusing ansible playbook. Using ansible we can run the tasks on mulitple targets server, it will be easy, error free and saves much time when compared to performing this setups manually in multiple target servers.

Thank you!!!
