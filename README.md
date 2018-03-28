![Udacity Linux Configuration Project](header.png)

# **Project Description**

Take a baseline installation of a Linux distribution on a virtual machine and prepare it to host your web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

# **Server Information**
**Public IP:** 35.160.162.145

**SSH Port:** 2200

**URL:** [http://35.160.162.145](http://35.160.162.145)

# **Steps to Complete Project**

## __Create Ubuntu Server Instance__

1. Create **Amazon Lightsail** instance with base Ubuntu install.

2. Download private SSH key generated by Lightsail and save to `~/.ssh`.

3. Connect using `ssh`.

    ```
    ssh ubuntu@35.160.162.145 -i ~/.ssh/private-key.pem
    ```

    **Note**: Before connecting on MacOS, it is necessary to change the file permissions of `private-key.pem`.

    ```
    sudo chmod 700 ~/.ssh/private-key.pem
    ```

## __Secure Ubuntu Server Instance__

Do the following steps while connected to server as `ubuntu`.

1. Update list of available package versions and upgrade all packages to latest version.

    ```
    sudo apt-get update
    sudo apt-get upgrade
    ```

2. Change SSH port from 20 to 2200, disable root login, and disable text password authentication by editing `/etc/ssh/sshd_config`.

    ```
    sudo nano /etc/ssh/sshd_config
    
    # Change the following lines accordingly...
    # Port 20 => Port 2200
    # PermitRootLogin without-password => PermitRootLogin no
    # [Uncomment] PasswordAuthentication no

    sudo service ssh restart 
    ```

**Warning:** Do not run the following commands before changing the SSH port to 2200. If you mistakenly disconnect from the server with the firewall enabled to block all incoming connections except those on port 2200 and SSH isn't configured to be listening on that port, you will effectively lock yourself out of the server.

3. Configure and enable firewall.

    ```
    sudo ufw status
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp
    sudo ufw allow www
    sudo ufw allow ntp
    sudo ufw enable
    sudo ufw status
    ```

### __Create `'grader'` User Account__

Do the following steps while connected to server as `ubuntu`.

1. Create a user named `grader` on the instance.

    ```
    sudo adduser grader
    ```

    **Note:** The password created for this user will never be used.

2. Give `grader` permission to `sudo` by adding a file named `grader` to  `/etc/sudoers.d` by executing...

    ```
    sudo nano /etc/sudoers.d/grader
    ```

    ... and then adding the following line to it and saving.

    ```
    grader ALL=(ALL) NOPASSWD:ALL
    ```

3. On your **local machine** in a linux terminal, generate an SSH key-pair named `grader-key` that will be used to authenticate SSH connections for user `grader`.

    ```
    cd ~/.ssh
    ssh-keygen
    ```

    The files `grader-key` and `grader-key.pub` will be created in the directory you executed the command.

4. Open `grader-key.pub` in a text editor and copy it's contents to the clipboard.

5. While connected to the server as `ubuntu`, switch to user `grader`.

    ```
    sudo su - grader
    ```

6. Make a directory named `.ssh` in the home directory of `grader` and change into it.

    ```
    mkdir ~/.ssh
    cd ~/.ssh
    ```

7. Create the file `authorized_keys` and paste the contents of the public key into it.

    ```
    sudo nano authorized_keys
    ```

8. Set file permisssions.

    **Note:** This is an important step, doing this incorrectly will cause troubles when trying to connect.

    ```
    sudo chown grader:grader /home/grader/.ssh
    sudo chmod 700 /home/grader/.ssh
    sudo chmod 644 /home/grader/.ssh/authorized_keys
    ```

9. Test that you've set everything up correctly by attempting to connect from your local machine as `grader` from a linux terminal.

    ```
    ssh grader@35.160.162.145 -p 2200 -i ~/.ssh/grader-key
    ```

    If you are unable to connect, you will have to login as `ubuntu` and switch into `grader` (step 5 above) and troubleshoot what went wrong.
    
    Possible causes:
    - You incorrectly set or forgot to set file permissions.
    - You copied the wrong key (public/private) into `authorized_keys`.

### __Prepare Project for Deployment__

Do the following steps while connected to server as `ubuntu`.

1. Install packages needed for deployment.

    ```
    sudo apt-get install apache2
    sudo apt-get install libapache2-mod-wsgi
    sudo apt-get install python-setuptools
    sudo apt-get install postgresql
    sudo apt-get install git
    sudo service apache2 resart
    ```

2. Create PostgreSQL database named `catalog`.

    1. Switch to `postgres` user account.

        ```
        sudo su - postgres
        ```

    2. Open `psql` interactive console.

        ```
        psql
        ```

    3. While in `psql`, create a database named 'catalog' with the following SQL query.

        ```
        CREATE DATABASE catalog;
        ``` 

3. Create user account `www-data` that Apache will use to connect to the database.

    ```
    sudo adduser www-data
    ```

4. Set local timezone to UTC.

    ```
    sudo timedatectl set-timezone UTC
    ```

### __Deploy Item Catalog Project__

Do the following steps while connected to server as `ubuntu`.

1. Enable `mod-wsgi` module on Apache.

    ```
    sudo a2enmod wsgi
    ```

2. Change into the directory `/var/www` and clone the `catalog` project from GitHub into it.

    ```
    cd /var/www
    sudo git clone https://github.com/item-catalog catalog
    ```

3. Install Python project dependcies.

    ```
    cd catalog
    pip install -r requirements.txt
    ```

4. Create a new file named `catalog.wsgi` in `/var/www/catalog`.

    ```
    sudo nano catalog.wsgi
    ```

    `catalog.wsgi` should contain the following code.

    ```
    !/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, '/var/www/catalog')
    from main import APP as application
    ```

5. Create a configuration file for Apache to tell `mod-wsgi` how to load your application and which incoming traffic to route to it.

    ```
    cd /etc/apache2/sites-available
    sudo nano catalog.conf
    ```
    `catalog.conf` should contain the following...

    ```
    <VirtualHost *:80>
        ServerName http://35.160.162.145/
        ServerAdmin john@johnarchuletta.com
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi
        <Directory /var/www/catalog>
                Order allow,deny
                Allow from all
        </Directory>
        Alias /static /var/www/catalog/static
        <Directory /var/www/catalog/static/>
                Order allow,deny
                Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        DocumentRoot /var/www/catalog
    </VirtualHost>
    ```

6. Originally, my `item-catalog` project used `sqlite` as it's database, so it was necessary to go back into my code and change a few things. Mainly, I had to alter my connection strings to use Postgres and also install a few other dependencies needed by `sqlalchemy` to interact with Postgres.

7. Initalize project database.

    ```
    cd /var/www/catalog/db
    python db_setup.py
    ```

8. Restart Apache.

    ```
    sudo service apache2 restart
    ```
## All done!

The item catalog should now be accessible at the URL listed at the top of this document!
