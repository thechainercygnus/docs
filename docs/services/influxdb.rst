========
InfluxDB
========

.. meta::
   :description lang=en: Install and configure InfluxDB, with some example use cases.

InfluxDB is a time-series database that is great for storing metrics or other time driven data. This installs OSS v1.8.5.

Installation notes
-----


Virtual Machine Configuration
-----------------------------

.. code-block:: shell-session

   brycej@shioxhast:~$ sudo virt-install --name influxdb \
   --network mac=00:0C:53:00:00:01 \
   --ram=2048 \
   --disk size=100 \
   --vcpus 2 \
   --os-type linux \
   --os-variant ubuntu20.04 \
   --graphics none \
   --location 'http://archive.ubuntu.com/ubuntu/dists/focal/main/installer-amd64/' \
   --extra-args "console=tty0 console=ttyS0,115200n8"

Now we download the tarball and install the service

.. code-block:: shell-session

   brycej@influxdb:~$ sudo apt update && sudo apt upgrade -y
   brycej@influxdb:~$ wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
   brycej@influxdb:~$ source /etc/lsb-release
   brycej@influxdb:~$ echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
   brycej@influxdb:~$ sudo apt update && sudo apt install influxdb -y
   brycej@influxdb:~$ sudo systemctl enable --now influxdb
   brycej@influxdb:~$ sudo sudo systemctl enable --now influxdb

   [http]
   # Determines whether HTTP endpoint is enabled.
   enabled = true

   # Determines whether the Flux query endpoint is enabled.
   flux-enabled = true

   # The bind address used by the HTTP service.
   bind-address = ":8086"

   brycej@influxdb:~$ sudo ufw allow 8086/tcp
   brycej@influxdb:~$ sudo ufw enable
   brycej@influxdb:~$ influx
   Connected to http://localhost:8086 version 1.8.5
   InfluxDB shell version: 1.8.5
   > CREATE USER admin WITH PASSWORD '<password>' WITH ALL PRIVILEGES
   > CREATE DATABASE crypto
   > CREATE DATABASE servers
   > exit
   