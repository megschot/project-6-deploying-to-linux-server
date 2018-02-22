# Linux Server Project
Public IP: 18.222.60.55
<br/>Server OS: Ubuntu 16.04 LTS
<br/>SSH Port: 2200
<br/>Server Provider: Amazon Lightsail
<br/>URL: http://18.222.60.55/

## 1 - Setup your server.
1. Start a new Ubuntu Linux server instance on Amazon Lightsail.
2. Follow the instructions provided to SSH into your server.
3. Update all currently installed packages. <br/>
    ```
    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get install finger

## 2 - Give grader access and configure authentication.
In order for your project to be reviewed, the grader needs to be able to log in to your server.

4. Create a new user account named grader. <br/>
          ``` sudo adduser grader ```
5. Give grader the permission to sudo. <br/>
     ``` sudo nano /etc/sudoers.d/grader ``` <br/>
          ```add this line and save: grader ALL=(ALL:ALL) ALL ```
6. Create an SSH key pair for grader using the ssh-keygen tool.
    * On your local machine: <br/>
        ```ssh-keygen -f ~/.ssh/proj6_key.rsa```
    * In the VM as ubuntu user: <br/>
        ```
        sudo mkdir /home/grader/.ssh
        sudo touch /home/grader/.ssh/authorized_keys
        sudo nano /home/grader/.ssh/authorized_keys
            * paste the contents from the proj6_key.pub file you saved on your local machine and save
        sudo chmod 700 /home/grader/.ssh
        sudo chmod 644 /home/grader/.ssh/authorized_keys
        sudo chown -R grader:grader /home/grader/.ssh
        ```
    * Now you can login to the remove VM through ssh as grader <br/>
        ``` ssh -i ~/.ssh/proj6_key.rsa grader@18.222.60.55 ```
7. sudo nano /etc/ssh/sshd_config: <br/>
    ```
    * Modify: PermitRootLogin no
    * Modify: PasswordAuthentication no
    * Add new line: AllowUsers grader ubuntu
    ```
8. Restart ssh service <br/>
    ``` sudo service ssh restart ```

## 3 - Secure Server
9. Change the SSH port from 22 to 2200. Make sure to configure the Lightsail firewall to allow it. <br/>
    ``` nano /etc/ssh/sshd_config --> modify Port 22 to Port 2200 ```
    * On Amazon Lightsail's "Networking" tab, add "Custom - TCP - 2200" to the firewall connections. <br/>
    ``` sudo service ssh restart``` 
    * You can now login with this command: <br/>
        ``` ssh -i ~/.ssh/proj6_key.rsa -p 2200 grader@18.222.60.55 ```
10. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123). <br/>
    ```
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp
    sudo ufw allow 80/tcp
    sudo ufw allow 123/udp
    sudo ufw enable
    ```

## 4 - Prepare to deploy your project.
9. Configure the local timezone to UTC. <br/>
    ``` sudo timedatectl set-timezone UTC ```
10. Install and configure Apache to serve a Python mod_wsgi application. <br/>
    ```
    sudo apt-get install apache2
    sudo apt-get install libapache2-mod-wsgi-py3
    sudo a2enmod wsgi
    sudo service apache2 start
    ```
11. Install git <br/>
    ```
    sudo apt-get install git
    git config --global user.name <username>
    git config --global user.email <email>
    ```
12. Install and configure PostgreSQL: <br/>
    ```sudo apt-get install postgresql```
    * Make sure psql only allows connections from localhost <br/>
        ```sudo nano /etc/postgresql/9.5/main/pg_hba.conf``` <br/>
        * localhost (127.0.0.1 for IPv4 or ::1 for IPv6), no external connections <br/>
    ``` 
    sudo su - postgres
    psql
    CREATE USER catalog WITH PASSWORD 'catalog';
    ALTER USER catalog CREATEDB;
    CREATE DATABASE catalog OWNER catalog;
    REVOKE ALL ON SCHEMA public FROM public;
    GRANT ALL ON SCHEMA public TO catalog;
    \q
    exit
    ```

## 5- Deploy the Item Catalog project.
13. Clone and setup your Item Catalog project from the Github repository you created earlier in this Nanodegree program. <br/>
    ```
    cd /var/www
    sudo mkdir catalog
    sudo chown -R grader:grader catalog
    cd catalog
    git clone https://github.com/megschot/project-4-catalog.git catalog
    ```
14. Modifications to the git repository <br/>
    ```
    * database_setup.py
        - engine = create_engine('sqlite:///catalog.db')
        + engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
    * test_data.py
        - engine = create_engine('sqlite:///catalog.db')
        + engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
    * application.py
        - CLIENT_ID = json.loads(open('client_secrets.json', 'r').read())['web']['client_id']
        + CLIENT_ID = json.loads(
            open('/var/www/catalog/catalog/client_secrets.json', 'r').read())['web']['client_id']
        - engine = create_engine('sqlite:///catalog.db')
        + engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
        - oauth_flow = flow_from_clientsecrets('client_secrets.json', scope='')
        + oauth_flow = flow_from_clientsecrets('/var/www/catalog/catalog/client_secrets.json', scope='')
        * move app.secret_key = 'super_secret_key' outside of if __name__ = '__main__'
        - app.run(host='0.0.0.0')
        + app.run()
    * client_secrets.json
        * Login to Google Developers Console <https://console.developers.google.com/apis/credentials>, add http://18.222.60.55 to Authorized JavaScript origins.
        * Update in client_secrets.json
15. Setup the virtual machine <br/>
    ```
    sudo apt-get install python3 python3-pip
    sudo pip3 install --upgrade pip
    sudo pip3 install virtualenv
    cd /var/www/catalog/catalog/
    virtualenv venv --python=python3
    source venv/bin/activate
    sudo chmod -R 777 venv
    ```
    * pip3 install the following:
        * flask
        * sqlalchemy
        * httplib2
        * oauth2client
        * requests
        * psycopg2
    * Initiate database <br/>
        ```
        python3 test_data.py
        python3 application.py # ctrl+c to get out of initializing
        ```
    *Deactivate virtual environment <br/>
        ``` deactivate ```
16. Create a new wsgi file <br/>
   ``` sudo nano /var/www/catalog/catalog/catalog.wsgi ```
    * add the following and save: <br/>
        ```
        #! /usr/bin/env python
        import sys
        sys.path.insert(0,"/var/www/catalog/catalog")
        from application import app as application
        ```
17. Create New Configuration File <br/>
    ``` sudo nano /etc/apache2/sites-available/catalog.conf ```
    * Add the following:
        ```
          <VirtualHost *:80>
            ServerName 18.222.60.55
            ServerAlias ec2-18-222-60-55.us-east-2@compute.amazonaws.com
            ServerAdmin admin@18.222.60.55
            WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python3.5/site-packages
            WSGIProcessGroup catalog
            WSGIScriptAlias / /var/www/catalog/catalog/catalog.wsgi
            <Directory /var/www/catalog/catalog/>
            Order allow,deny
            Allow from all
            </Directory>
            Alias /static /var/www/catalog/catalog/static
            <Directory /var/www/catalog/catalog/static/>
            Order allow,deny
            Allow from all
            </Directory>
            ErrorLog ${APACHE_LOG_DIR}/error.log
            CustomLog ${APACHE_LOG_DIR}/access.log combined
          </VirtualHost>
         ```
     ```
     sudo a2ensite catalog
     sudo apache2ctl restart
     ```
