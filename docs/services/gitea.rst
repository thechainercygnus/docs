Gitea
=====

.. meta::
   :description lang=en: Installation and Configuration of Gitea

Installation notes
-----

These are the steps I went through to get Gitea installed to my preferrences on a KVM using libvirtd. Future plans involve automating this entire process in Ansible.

Preparing Your Environment
--------------------------

The first thing I like to do for each new service deployment is get DNS ready. I like using friendly names wherever possible. Since I will be defining the MAC address of my VM manually, I can pre-set a DHCP reservation and create my A Record. Since I use Pi-Hole for both DHCP and DNS, this is a simple task in the admin interface. Reference the network mac line under Virtual Machine Configuration.


Virtual Machine Configuration
-----------------------------

The following walks through a net install of Ubuntu 20.04. Modify vcpus ram and disk size as needed. Walk through configuration options as required, being sure to install OpenSSH Server when package selection occurs. This walkthrough presupposes you use the same hostname for your intial user as you do on your primary workstation. If you prefer to use different names, please be sure to prepend usernames in all ssh commands.

.. code-block:: shell-session
    :linenos:

    {user}@{host} $ sudo virt-install --name gitea \
        --network mac=00:0C:53:00:00:04 \ # Make sure you match your DNS / DHCP entries
        --ram=1024 \
        --disk size=100 \
        --vcpus 1 \
        --os-type linux \
        --os-variant ubuntu20.04 \
        --graphics none \
        --location 'http://archive.ubuntu.com/ubuntu/dists/focal/main/installer-amd64/' \
        --extra-args "console=tty0 console=ttyS0,115200n8"

Copy your ssh-key to the new VM and verify you can access.

.. code-block:: shell-session
    :linenos:

    {user}@{host} $ ssh-copy-id {yourhostname}
    /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
    /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
    {user}@{yourhostname}'s password:

    Number of key(s) added: 1

    Now try logging into the machine, with:   "ssh '{yourhostname}'"

    {user}@{host} $ ssh {yourhostname}

You should now be logged into your Virtual Machine via SSH. Stay in this remote session for the remaining tasks.

Application Environment Configuration
-------------------------------------

Install git and create the git user that will run Gitea.

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo apt update && sudo apt install git -y
    {user}@gitea $ sudo adduser \
        --system \
        --shell /bin/bash \
        --gecos 'Git Version Control' \
        --group \
        --disabled-password \
        --home /home/git \
      git

Install PostgreSQL and configure for use

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo apt install postgresql -y
    {user}@gitea $ sudo su postgres
    postgres@gitea $ psql
    postgres=# CREATE USER gitea WITH PASSWORD '<password>';
    CREATE ROLE
    postgres=# CREATE DATABASE gitea OWNER gitea;
    CREATE DATABASE
    postgres=# \q
    postgres@gitea $ exit

Now we can use the git user we created before to download and run Gitea for initial configuration.

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo su git
    git@gitea $ cd ~
    git@gitea $ mkdir gitea
    git@gitea $ cd gitea
    git@gitea $ wget -O gitea https://dl.gitea.io/gitea/1.14.1/gitea-1.14.1-linux-amd64
    git@gitea $ chmod +x gitea
    git@gitea $ ./gitea web
    # Run through web setup at {yourhostname}:3000
    {user}@gitea $ exit
    {user}@gitea $ sudo nano /etc/systemd/system/gitea.service

Add the following to the gitea.service file:

.. code-block:: shell-session
    :linenos:

    [Unit]
    Description=Gitea (Git with a cup of tea)
    After=syslog.target
    After=network.target
    After=postgresql.service

    [Service]
    RestartSec=2s
    Type=simple
    User=git
    Group=git
    WorkingDirectory=/home/git/gitea
    ExecStart=/home/git/gitea/gitea web
    Restart=always
    Environment=USER=git HOME=/home/git

    [Install]
    WantedBy=multi-user.target

Save the file and now we can start the service. Verify you can access the web interface at {yourhostname}:3000 after these steps. 

.. code-block:: shell-session
    :linenos:

    {user}@gitea sudo systemctl enable gitea.service
    {user}@gitea sudo systemctl start gitea.service

Install and Configure Nginx
---------------------------

Install Nginx and create a new sites-enabled file for the Gitea.

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo apt install nginx -y
    {user}@gitea $ sudo nano /etc/nginx/sites-enabled/gitea

    server {
        listen 80;
        server_name {yourhostname};

        location / {
            proxy_pass http://localhost:3000;
        }

        proxy_set_header X-Real-IP $remote_addr;
    }

For sanitary purposes let's remove the default site and then we can reload nginx. Once this is done, we can access gitea by visting http://{yourhostname} now.

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo rm /etc/nginx/sites-enabled/default
    {user}@gitea $ sudo service nginx reload

Install and Configure fail2ban
------------------------------

This is one of those things that I think you just do, right? Anyways, I'm not planning on exposing my git server to the wide world, but I may eventually grant remote access to someone for a project, in which case I might as well have some protection in place right? Plus It's fun to learn new tools.

As usual, install and configure

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo apt install fail2ban -y
    {user}@gitea $ sudo nano /etc/fail2ban/filter.d/gitea.conf

    [Definition]
    failregex =  .*Failed authentication attempt for .* from <HOST>
    ignoreregex =

    {user}@gitea $ sudo nano /etc/fail2ban/jail.d/jail.local

    [gitea]
    enabled = true
    port = http,https
    filter = gitea
    logpath = /home/git/gitea/log/gitea.log
    maxretry = 10
    findtime = 3600
    bantime = 900
    action = iptables-allports

And the usual restart after configuration changes.

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo service fail2ban restart

Generating an SSL Certificate
-----------------------------

Even if I am only hosting internally for my own usage, I like knowing my traffic is encrypted. And for things like this, the best tool I have found is to just use certbot itself to generate the cert. We also will need to store our Cloudflare API Token somewhere accessible.

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo apt install software-properties-common snapd -y
    {user}@gitea $ sudo snap install certbot --classic
    {user}@gitea $ sudo ln -s /snap/bin/certbot /usr/bin/certbot
    {user}@gitea $ sudo snap set certbot trust-plugin-with-root=ok
    {user}@gitea $ sudo snap install certbot-dns-cloudflare

Now we have to set up the environment real quick. Should I set permissions on cloudflare.ini differently? Check certbot docs

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ mkdir .secrets
    {user}@gitea $ mkdir .secrets/certbot
    {user}@gitea $ nano .secrets/certbot/cloudflare.ini 

    dns_cloudflare_api_token = 0123456789abcdef0123456789abcdef01234567

Now we can generate the certificate.

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ certbot certonly \
    {user}@gitea $ --dns-cloudflare \
                   --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini \
                   -d {yourhostname}


Reconfigure Nginx to use SSL
----------------------------

Not that we have our SSL Certificate we can reconfigure Nginx to use SSL.

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo nano /etc/nginx/sites-enabled/gitea
    
    server {
        listen 443;

        ssl on;
        ssl_certificate /etc/letsencrypt/live/{yourhostname}/fullchain.pem;
        ssl_certificate_key  /etc/letsencrypt/live/{yourhostname}/privkey.pem;

        server_name {yourhostname};
        
        location / {
            proxy_pass http://localhost:3000;
        }

        proxy_set_header X-Real-IP $remote_addr;
    }

    {user}@gitea $ sudo service nginx reload

Configure Certificate Auto-Renewal
----------------------------------

The only problem with LE Certs is that they have short expirations. 3 months, to be exact. So we can configure the system to maintain it's own certificate.

.. code-block:: shell-session
    :linenos:

    {user}@gitea $ sudo nano /etc/systemd/system/certbot-renewal.service

    [Unit]
    Description=Certbot Renewal

    [Service]
    ExecStart=/usr/bin/certbot renew

    {user}@gitea $ sudo nano /etc/systemd/system/certbot-renewal.timer

    [Unit]
    Description=Timer for Certbot Renewal

    [Timer]
    OnBootSec=300
    OnUnitActiveSec=1d

    [Install]
    WantedBy=multi-user.target

    {user}@gitea $ sudo systemctl enable certbot-renewal.timer
    {user}@gitea $ sudo systemctl start certbot-renewal.timer

In admin go to System Administration and run the `Update the '.ssh/authorized_keys' file with Gtea SSH keys.` operation.

Done