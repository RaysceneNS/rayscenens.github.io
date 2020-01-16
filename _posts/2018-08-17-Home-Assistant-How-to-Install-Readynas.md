---
tags: [ReadyNAS]
---

# How To Install Home Assistant on a Netgear ReadyNAS

A quick reference guide for how to perform the necessary steps to install homeassistant on a NetGear ReadyNAS.
The purpose of this guide is to record the steps required to install Home Assistant on a NetGear ReadyNAS running OS 6.9.3.

## Install Python 3.5.6

Open an SSH terminal and login to your ReadyNAS, you will first need to enable the SSH feature within the System - Services area of the ReadyNAS admin panel.

Home assistant requires a minimum python version of 3.5.6, as of this writing apt-get only makes 3.4.2 available on this system. To get around this issue we need to roll our own python build.

To install the python 3.5.6 from source you will need a working GCC build environment first. The following apt-get will install the libraries required to build python3 as well as load development dependencies for python such as sqlite3.

```Bash
apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev
libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev
xz-utils tk-dev libffi-dev liblzma-dev
```

Next download the source for python 3.5.6 so we can build it.
*Note* the switch to enable sqlite3 extension support on the configure line

```Bash
mkdir /root/src/
cd /root/src/
wget http://www.python.org/ftp/python/3.5.6/Python-3.5.6.tgz
gzip -d Python-3.5.6.tgz
tar xf Python-3.5.6.tar
cd Python-3.5.6
./configure --enable-loadable-sqlite-extensions
make
make install
```

On my machine the make install places the output in /usr/local/bin

Check that python is installed correctly with this command.

```Bash
python3.5 --version
```

## Create a virtual environment to host Home Assistant

Enter the following commands to create a virtual python environment in the /apps directory, then install wheel and homeassistant in this folder.

```Bash
$ cd /apps
$ python3.5 -m venv homeassistant
$ cd homeassistant
$ source bin/activate
$ python3 -m pip install wheel
$ python3 -m pip install homeassistant
```

Finally check that homeassistant is running by launching it now.

```Bash
$ hass
```

## Autostart service via systemd

It is recommended to create a system user to launch the service as.

place this file in /etc/systemd/system/home-assistant@[your user].service

```Bash
[Unit]
Description=Home Assistant
After=network-online.target

[Service]
Type=simple
User=%i
ExecStart=/apps/homeassistant/bin/hass -c "/home/[your user]/.homeassistant"

[Install]
WantedBy=multi-user.target
```

You need to reload systemd to make the daemon aware of the new configuration.

```Bash
$ sudo systemctl --system daemon-reload
```

To have Home Assistant start automatically at boot, enable the service.

```Bash
$ sudo systemctl enable home-assistant@[your user]
```

To start Home Assistant, use this command.

```Bash
$ sudo systemctl start home-assistant@[your user]
```