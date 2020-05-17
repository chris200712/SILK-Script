#!/bin/bash

# (***)Downloads of NetSA software can be found on our website: http://tools.netsa.cert.org/. (***)
# (***)The versions shown here are current as of this writing.(***)
# Modified on Feb 1 2020 by SSG Frazier Christopher C. 25D
#
# Make sure to have offline Repo running via VMWare workstation to download dependancies needed.  Files are on Lacy drive
# Developement tools are needed for YAF
# If things start failing when running this script...you know it is dependancy problems.  
#
# This script will configure everything needed to run SiLK
# When the script is finished run:
#					  IMPORTANT: Make sure the interface (eth0 below) matches the interface on which you want to capture.
#
#						sudo nohup /usr/local/bin/yaf --silk --ipfix=tcp --live=pcap  --out=127.0.0.1 \
#					  --ipfix-port=18001 --in=eth0 --applabel --max-payload=384 &
#							Ping -c 4 x.x.x.x  (wait for 10 minutes)
#								sudo ps -auxww | grep yaf  (If YAF is not running...install dependancies and rerun script)
#								sh /etc/init.d/rwflowpack status
#								cat /var/log/rwflowpack-*.log
#							Query:
#								/usr/local/bin/rwfilter --sensor=S0 --proto=0-255 --pass=stdout --type=all | rwcut | tail
#										***This does survive reboot***
cd ~
mkdir tmp
cd tmp
wget http://tools.netsa.cert.org/releases/silk-3.19.0.tar.gz
wget http://tools.netsa.cert.org/releases/libfixbuf-2.4.0.tar.gz
wget http://tools.netsa.cert.org/releases/yaf-2.11.0.tar.gz
wget http://tools.netsa.cert.org/releases/analysis-pipeline-5.11.3.tar.gz
#wget http://tools.netsa.cert.org/releases/isilk-0.6.2.tar.gz  ##WINDOWS GUI
wget http://tools.netsa.cert.org/releases/rayon-1.4.3.tar.gz



#(***)REQUIRED TOOL DEPENDENT LIBRARIES(***)
# ***Silk***
# You will need to get:
# gcc , gcc-c++, glib2, glib2-dev, libpcap, libpcap-dev, python and python-dev. Refer to: https://tools.netsa.cert.org/silk/silk-install-handbook.html

# ***YAF*** 
# As of YAF v1.0 the libairframe libraries come packaged with YAF 
# and you do not need to perform a separate install of the library.
# You will need to get:
# glib2, libpcap, libfixbuf, groupinstall "Development Tools", install libpcap libpcap-devel pcre pcre-devel glib2-devel

# ***Fixbuf***
# glib2

# ***Analysis-Pipeline***
# SILK, schemaTools, fixbuf, snarf

# ***iSiLK***
# rayon
# Python Modules: wxPython2.8, NumPy 1.2.0 or higher, MatPlotLib 0.98.3 or higher,
# Paramiko 1.7.2, Pycrypto 2.0.1.win32-py2.4

sudo yum -y install glib2.0 libglib2.0-dev libpcap-dev g++ 
sudo yum -y install python-numpy python-dev python-wxgtk2.8 python-matplotlib
sudo yum -y install python-paramiko python-pycryptopp python-cairo libcairo2-dev

# ***Verify that you can run Python and load these modules***
#python
#import wx 
#import matplotlib
#import numpy
#import paramiko
#exit()


# (***)INSTALL TOOLS(***)

# ***Install Fixbuf***

cd ~/tmp
tar -zxvf libfixbuf-2.4.0.tar.gz
cd libfixbuf-2.4.0
./configure && make
sudo make install

# ***Install YAF***

cd ~/tmp
tar -zxvf yaf-2.11.0.tar.gz
cd yaf-2.11.0
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig
./configure --enable-applabel
make
sudo make install

# *** Install Rayon ***
cd tmp
tar xzf rayon-1.4.3.tar.gz
cd rayon-1.4.3
python setup.py help    # to see options
python setup.py install # to install

# *** Install Ananlysis ***
tar zxf analysis-pipeline-5.11.3.tar.gz
cd analysis-pipeline-5.11.3
./configure --prefix=/usr
make
make install

# ***Install Silk***
# Make a new directory to store our data

sudo mkdir /SiLK

# ***Build and install SiLK***

cd ~/tmp
tar -xvzf silk-3.19.0.tar.gz
cd silk-3.19.0
./configure --enable-data-rootdir=/SiLK --with-libfixbuf=/usr/local/lib/pkgconfig/ --with-python --enable-ipv6
make
sudo make install

# Add the following paths to ld.so.conf 
# to avoid exporting LD_LIBRARY_PATH each time you use SiLK

cat <<EOF >>silk.conf
/usr/local/lib
/usr/local/lib/silk
EOF
sudo mv silk.conf /etc/ld.so.conf.d/

# run ldconfig

sudo ldconfig



# (***)CONFIGURE HOST FIREWALL TO ALLOW YAF TO TALK TO rwflowpack BY ALLOWING 18001 IN.(***)

#sudo ufw allow proto tcp from 127.0.0.1 to 127.0.0.1 port 18001



# (***)CONFIGURE SiLK(***)

cd ~/tmp/silk-3.19.0
sudo cp site/twoway/silk.conf /SiLK

# ***Create sensors.conf file***
# Edit "group my-network" with desired (Internal) IP ranges

cat <<EOF >sensors.conf
probe S0 ipfix
 listen-on-port 18001
 protocol tcp
 listen-as-host 127.0.0.1
end probe
group my-network
 ipblocks 10.25.0.0/24 # address of eth0. CHANGE THIS.
# ipblocks 10.0.0.0/8 # other blocks you consider internal
end group
sensor S0
 ipfix-probes S0
 internal-ipblocks @my-network
 external-ipblocks remainder
end sensor
EOF
sudo mv sensors.conf /SiLK

LD_LIBRARY_PATH=/usr/local/lib /usr/local/bin/python

# ***INSTALL iSiLK***  WINDOWS ONLY
#cd ~/tmp
#tar -zxvf isilk-0.6.2.tar.gz
#cd isilk-0.6.2
#./configure && make
#sudo make install


# (***)CONFIGURE rwflowpack(***)
# configure rwflowpack to listen for flows from YAF. 
# We copy the default rwflowpack.conf, changing some values.

cat /usr/local/share/silk/etc/rwflowpack.conf | \
sed 's/ENABLED=/ENABLED=yes/;' | \
sed 's/SENSOR_CONFIG=/SENSOR_CONFIG=\/SiLK\/sensors.conf/;' | \
sed 's/SITE_CONFIG=/SITE_CONFIG=\/SiLK\/silk.conf/' | \
sed 's/LOG_TYPE=syslog/LOG_TYPE=legacy/' | \
sed 's/LOG_DIR=.*/LOG_DIR=\/var\/log/' | \
sed 's/CREATE_DIRECTORIES=.*/CREATE_DIRECTORIES=yes/' \
>> rwflowpack.conf
sudo mv rwflowpack.conf /usr/local/etc/

# ***Copy start up script into /etc/init.d and sit it to start on boot.***

sudo cp /usr/local/share/silk/etc/init.d/rwflowpack /etc/init.d
cd /etc/init.d/
sudo sudo chkconfig --add rwflowpack start 20 3 4 5 .
sudo service rwflowpack start




