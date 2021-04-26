Certbot
=======

Certbot is a handy tool for obtaining and managing SSL certificates from Let's Encrypt.

Installation
------------

Notes on how to install

Configuration
-------------

https://certbot.eff.org/docs/using.html#using-ecdsa-keys

.. code-block:: shell-session

    brycej@gitea:~$ wget https://url.to.git/repo/config/certbot.ini
    brycej@gitea:~$ sudo mv certbot.ini /etc/letsencrypt/

    # Use ECC for the private key
    key-type = ecdsa
    elliptic-curve = secp384r1

    # Use a 4096 bit RSA key instead of 2048
    rsa-key-size = 4096

    # Uncomment and update to register with the specified e-mail address
    email = foo@example.com

    # Uncomment to automatically agree to the terms of service of the ACME server
    agree-tos = true

More changes?