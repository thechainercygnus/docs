Grafana
=======

Virtual Machine
---------------

Installation
------------

.. code-block:: shell-session
    bjenkins@grafana:~$ sudo apt update 
    bjenkins@grafana:~$ sudo apt install apt-transport-https -y
    bjenkins@grafana:~$ sudo apt install software-properties-common wget -y
    bjenkins@grafana:~$ wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
    bjenkins@grafana:~$ echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
    bjenkins@grafana:~$ sudo apt update
    bjenkins@grafana:~$ sudo apt install grafana