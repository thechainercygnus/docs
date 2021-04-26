Standard VM Configuration
=========================

Hostnames
---------


Configure OpenSSH
-----------------

This is some standard OpenSSH configuration that I push to all servers.

.. code-block:: shell-session

    brycej@guest:~$ sudo nano /etc/ssh/sshd_config
    PermitRootLogin no
    PasswordAuthentication no
    brycej@guest:~$ sudo /etc/init.d/ssh reload
    Reloading ssh configuration (via systemctl): ssh.service.

Configure UFW
-------------

Since I use UFW to restrict non-essential traffic, this is one of the first commands I run on a new server.

.. code-block:: shell-session

    brycej@guest:~$ sudo ufw allow OpenSSH
    Rules updated
    Rules updated (v6)
