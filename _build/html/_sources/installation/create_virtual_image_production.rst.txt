Create G-OnRamp Virtual Machine Image
===================================

Requirements
------------

1. Download and install Virtual Box => https://www.virtualbox.org
2. Download Ubuntu Server 16.04.1 LTS => https://www.ubuntu.com/download/server

Step by step instruction
-------------------------
1. Install Ubuntu server to the VirtualBox
******************************************
See `How to Install Ubuntu on VirtualBox <http://tecadmin.net/install-ubuntu-on-virtualbox/>`_

Additional settings when installing the Ubuntu

- Hostname: ubuntus
- Full name: galaxyadmin
- User name: galaxyadmin
- password: 1234
- Use entire disk and set up LVM
- No proxy configured
- Install security updates automatically
- Install LAMP server, PostgreSQL database, OpenSSH server
- Create new GRUB boot loader
  
2. Initial Ubuntu setup
***********************
Start the Ubuntu and login in to update the system.

Update packages:: 
 
  $ sudo apt-get update
  $ apt-get --with-new-pkgs upgrade
  
Install dependencies::

  $ sudo apt-get install build-essential
  $ sudo apt-get install cmake
  $ sudo apt-get install zlib1g-dev  
  $ reboot

3. Set up internet host-only adapter (used to connect guest VM from host by ssh)
********************************************************************************
Shutdown the virtual machine and add vboxnet0 to Adapter 2 as Host-only Adapter. Then restart the virtual machine. 

.. image:: network.png
    :width: 500px
    :align: center
    :alt: add vboxnet0 to Adapter 2 

On the host, type command::

$ ifconfig

Find the vboxnet0 ip::

    vboxnet0: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
    ether 0a:00:27:00:00:00
    inet 192.168.56.1 netmask 0xffffff00 broadcast 192.168.56.255

On the Ubuntu guest, list interfaces by typing the command::

$ ip addr

You should see three interfaces like lo, enp0s3, enp0s8. We will use the third. 

Edit the interfaces file by::

$ cd /etc/network/interfaces

Add following enp0s8 configuration to the file::

    auto enp0s8
    iface enp0s8 inet static
    address 192.168.56.11
    netmask 255.255.255.0

Then activate the interface::

$ sudo ifup enp0s8

Check if enp0s8 got correct address. You should see your ip by typing::

    $ ip addr show enp0s8
     ...
    inet 192.168.56.11/24 brd 192.168.56.255 scope global secondary enp0s8

If not correct, you may run::

$ sudo ifdown enp0s8 
$ sudo ifup enp0s8
$ reboot

Now you can access to Ubuntu guest through host by::

$ ssh galaxyadmin@192.168.56.11

4. Install galaxy
*****************
Create a NON-ROOT user called galaxy. Running as an existing user will cause problems down the line when you want to grant or restrict access to data.
$ sudo adduser galaxy
password: 2016
Add galaxy to sudo 
$ usermod -aG sudo galaxy
Then login with galaxy
$ su - galaxy

create user for ftp
$ sudo adduser galaxyftp
password: 2016

Make sure Galaxy is using a clean Python interpreter. Conflicts in $PYTHONPATH or the interpreter's site-packages/ directory could cause problems. Galaxy manages its own dependencies for the framework, so you do not need to worry about these. The easiest way to do this is with a virtualenv:
$ sudo apt-get update
$ sudo apt-get install python-pip
$ pip install --upgrade pip
$ sudo pip install virtualenv
$ virtualenv gonramp
$ source gonramp/bin/activate
 
Galaxy requires a few things to run: a virtualenv, configuration files and dependent python modules. Starting the server at the first time will set these thing up. 

Download Galaxy 17.01::

$ git clone -b release_17.01 https://github.com/galaxyproject/galaxy.git
# Rename galaxy folder
$ mv galaxy/ galaxy-dist

Configure Galaxy
/
[server:main]
host = 192.168.56.11
filter-with = proxy-prefix
cookie_path = /galaxy

[filter:proxy-prefix]
use = egg:PasteDeploy#prefix
prefix = /gonramp

debug = False
use_interactive = False
Disable filter-with = gzip
cleanup_job = onsuccess




Set up PostgreSQL database
**************************
$ sudo apt-get update
$ sudo apt-get install postgresql postgresql-contrib
$ sudo -u postgres createuser --superuser galaxy
$ sudo -u galaxy createdb galaxy
$ psql -u galaxy
$ \password 1234
In galaxy.init, set:
database_connection = postgresql://galaxy:1234@localhost/galaxy

Run Galaxy::

$ cd galaxy
$ sh run.sh

Set up proxy nginx
******************


# Install nginx from pre-build package. Not flexible
ref: https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-16-04#step-3-check-your-web-server

$ sudo systemctl stop apache2.service
$ sudo apt-get update
$ sudo apt-get install nginx
$ sudo apt-get install nginx-extras

# To remove nginx
# $ sudo apt-get remove nginx nginx-common # Removes all but config files.
# $ sudo apt-get purge nginx nginx-common # Removes everything.
# After using any of the above commands, use this in order to remove dependencies used by nginx which are no longer required.
# $ sudo apt-get autoremove 

# Install nginx from source
# ref: https://www.nginx.com/resources/admin-guide/installing-nginx-open-source/
# Install zlib
# $ wget http://zlib.net/zlib-1.2.11.tar.gz
# $ tar -zxf zlib-1.2.11.tar.gz
# $ cd zlib-1.2.11
# $ ./configure
# $ make
# $ sudo make install
# 
# Install openssl
# $ wget http://www.openssl.org/source/openssl-1.0.2f.tar.gz
# $ tar -zxf openssl-1.0.2f.tar.gz
# $ cd openssl-1.0.2f
# $ ./configure --prefix=/usr
# $ make
# $ sudo make install
# 
# Connecting to VM via FTP
# Check the network adapter 1. change it to "Bridged"
# $ reboot

# Common used commands
# ref: http://nginx.org/en/docs/beginners_guide.html
# nginx -s signal
# Where signal may be one of the following:
# stop — fast shutdown
# quit — graceful shutdown
# reload — reloading the configuration file
# reopen — reopening the log files
# example: nginx -s reload



#add server block, 
$ cd /etc/nginx/site-available
$ sudo vim galaxy
# add following server block to galaxy file
server {
    listen 80;
    root /var/www/html;
    # maximum file upload size
    client_max_body_size 10G;

    # pass most requests to the proxied Galaxy application
    location /gonramp {
        proxy_pass        http://192.168.56.11:8080;
        proxy_set_header    X-Forwarded-Host $host;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # directly serve static content in nginx
    location /gonramp/static {
        alias /home/galaxy/galaxy-dist/static;
        expires 24h;
    }
    location /gonramp/static/style {
        alias /home/galaxy/galaxy-dist/static/style/blue;
        expires 24h;
    }
    location /gonramp/static/scripts {
        alias /home/galaxy/galaxy-dist/static/scripts;
        expires 24h;
    }
    location /gonramp/robots.txt {
        alias /home/galaxy/galaxy-dist/static/robots.txt;
        expires 24h;
    }
}

# enable galaxy server
$ cd /etc/nginx/site-enabled
$ sudo ln -s /etc/nginx/site-available/galaxy

#Make sure that you either comment out or modify line containing default configuration for enabled sites. in /etc/nginx/nginx.conf, include /etc/nginx/sites-enabled/*;
$ cd /etc/nginx/site-enabled
$ rm default

# Change the root of nginx to /var/www/html/nginx, in order to distinct from apache page
# $ cd /etc/nginx/site-available
# $ sudo vim galaxy #change root to /var/www/html/nginx
# $ cd /var/www/html
# $ sudo mkdir /var/www/html/nginx
# $ mv index.nginx-debian.html nginx/
# $ sudo systemctl restart nginx

Rotate log files
****************

To use logrotate to rotate Galaxy log files, add a new file named "galaxy" to /etc/logrotate.d/ directory with something like:

PATH_TO_GALAXY_LOG_FILES {
  weekly
  rotate 8
  copytruncate
  compress
  missingok
  notifempty
}

6. Set up Galaxy
****************

Set proxy
#########

Galaxy application needs to be aware that it is running with a prefix (for generating URLs in dynamic pages). This is accomplished by configuring a Paste proxy-prefix filter in the [app:main] section of config/galaxy.ini and restarting Galaxy::
  
  [server:main]
    host = 192.168.56.11
  [filter:proxy-prefix]
    use = egg:PasteDeploy#prefix
    prefix = /gonramp
  [app:main]  
    filter-with = proxy-prefix
    cookie_path = /gonramp


Become an Admin
###############
In order to install tools, you have to become administrator for your Galaxy instance. First start the server, go to http://192.168.56.11:8080/gonramp, and register as a new user with your email address. 

#username: galaxyadmin@gonramp.org
# public name: galaxyadmin
#password: 12341234
  
Go to galaxy folder and find a sub-folder called config. Add a new file named galaxy.init in the config folder. You can copy the content of galaxy.init.sample into galaxy.init. In galaxy.init, search for the line containing “admin_users”. Add your user email address to admin users. (Replace None to your email address). You can add multiple admin users by appending another email and separating them with a comma::
  
  # this should be a comma-separated list of valid Galaxy users
  admin_users = galaxyadmin@gonramp.org
  
  
Set up conda
############

In galaxy.init, uncomment and edit the following conda configuration::

  conda_ensure_channels = conda-forge,r,bioconda,iuc
  conda_auto_install = True
  conda_auto_init = True
  
Connect your Galaxy to the test tool shed
#########################################

Galaxy is connected to the Main Tool Shed by default. Since some tools in G-OnRamp workflow are in the Test Tool Shed, you need to connect to the Test Tool Shed by modifying the “tool_sheds_conf.xml” in config folder::

  # Copy the “tool_sheds_conf.xml.sample” and rename it to “tool_sheds_conf.xml”
  $ cd /home/galaxy/config
  $ cp tool_sheds_conf.xml.sample tool_sheds_conf.xml
  # Open the file 
  $ vim tool_sheds_conf.xml

Uncomment the lines for the Test Tool Shed::
  
  <?xml version="1.0"?>
  <tool_sheds>
    <tool_shed name="Galaxy Main Tool Shed" url="https://toolshed.g2.bx.psu.edu/"/>
  <!-- Test Tool Shed should be used only for testing purposes.
    <tool_shed name="Galaxy Test Tool Shed" url="https://testtoolshed.g2.bx.psu.edu/"/> 
  -->
  </tool_sheds>
  
To::

  <?xml version="1.0"?>
  <tool_sheds>
    <tool_shed name="Galaxy Main Tool Shed" url="https://toolshed.g2.bx.psu.edu/"/>
    <tool_shed name="Galaxy Test Tool Shed" url="https://testtoolshed.g2.bx.psu.edu/"/> 
  </tool_sheds>
  
Add necessary datatypes
#######################

Copy the “datatypes_conf.xml.sample” and rename it to “datatypes_conf.xml”.
Add the line below in between <registration></registration>::

  <datatype extension="psl" subclass="True" type="galaxy.datatypes.tabular:Tabular" />

Other files should be ready (copy from .sample)
################################################
::

  $ cp dependency_resolvers_conf.xml.sample dependency_resolvers_conf.xml
  
Restart the server after you modified the configuration files. You can hit Ctrl-c to stop the server and then start again.

7. Install G-OnRamp tools
*************************

Go to Admin page, and click on Search Tool Shed. Click on the Tool Sheds to search and install. You can add all G-OnRamp tools in a separate panel section by adding a new tool panel section when you install the first tool and then add all the rest tools in the same panel. 

Click on Galaxy Main Tool Shed to install
#########################################

- ncbi_blast_plus (by devteam)
- Augustus 
- HISAT2
- StringTie 
- blastXmlToPsl 
- TrfBig 
- pslToBed 
- bamtobigwig (by yating-l)
- hubarchivecreator 
- multi_fasta_glimmer_hmm
- snap
- psltobigpsl (by yating-l)
- jbrowsearchivecreator
- gbtofasta
- regtools_junctions_extract (by yating-l)
- rename_scaffolds
- ucsc_blat
- ucsc_pslcdnafilter
- uscs_pslpostarget
- uscs_pslcheck

Tools need advanced configuration
#################################

1. multi_fasta_glimmer_hmm

Make a Dependencies folder in "/home/galaxy" and download Glimmer3 inside the folder by::

  $ mkdir Dependencies
  $ cd ~/Dependencies
  $ wget ftp://ccb.jhu.edu/pub/software/glimmerhmm/GlimmerHMM-3.0.4.tar.gz

Rename glimmerhmm executable file (glimmerhmm_linux_x86_64) to glimmerhmm and then add glimmerhmm to the path (in .bashrc).

You also need to use a trained organism by adding them as reference data in Galaxy:

Add the glimmer_hmm_trained_dir data table to tool_data_table_conf.xml in $GALAXY_ROOT/config/::

  <!-- glimmer_hmm trained_dir -->
  <table name="glimmer_hmm_trained_dir" comment_char="#">
    <columns>value, name, path</columns>
    <file path="tool-data/glimmer_hmm.loc" />
  </table> 
  
  

Add the glimmer_hmm.loc file referencing your trained organism, in tool-data. Uncomment the species and add the path to trained_dir. 

2. TrfBig

Because trf executable file is not renamed to trf successfully, so it needs to be renamed manually.
 
Go to installation path of trf::

  $ cd /home/galaxy/galaxy/database/dependencies/trf/4.07b/rmarenco/trfbig/e45bd0ffc1a4/bin
  $ mv trf407b.linux64 trf
  $ chmod 755 trf

3. Bam to Bigwig

libstdc++.so.6: version `GLIBCXX_3.4.20' not found, maybe because it is not using the right libstdc++.so.6. Delete the installed libstdc++.so.6 in the tool dependencies, so that the tool can use the system version of libstdc++.so.6::

  $ cd /home/galaxy/galaxy/database/dependencies/ucsc_tools/312/iuc/package_ucsc_tools_312/2d6bafd63401/lib/
  $ rm libstdc++.so.6



FTP server 
**********
ref: https://galaxyproject.org/admin/config/upload-via-ftp/

in the config file, galaxy.init, set
ftp_upload_dir = /home/galaxy/galaxy-dist/database/files/
ftp_upload_site = 192.168.56.11

Allow your FTP server to read Galaxy's database
===============================================
You'll need to grant a user access to read emails and passwords from the Galaxy database. Although the user Galaxy connects with could be used, I prefer to use a least-privilege setup wherein a separate user is created for the FTP server which has permission to SELECT from the galaxy_user table and nothing else. In postgres this is accomplished with:

postgres@dbserver% createuser -SDR galaxyftp
postgres@dbserver% psql galaxydb
Welcome to psql 8.X.Y, the PostgreSQL interactive terminal.

Type:  \copyright for distribution terms
       \h for help with SQL commands
       \? for help with psql commands
       \g or terminate with semicolon to execute query
       \q to quit

galaxydb=# ALTER ROLE galaxyftp PASSWORD 'dbpassword';
ALTER ROLE
galaxydb=# GRANT SELECT ON galaxy_user TO galaxyftp; 
GRANT

set up FTP server
=================
ref:https://kyup.com/tutorials/install-setup-ftp-server-proftpd/

$ sudo apt-get install proftpd
$ sudo apt-get install proftpd-mod-psql
$ sudo nano /etc/proftpd/proftpd.conf  

When we are ready with the configuration we can start up the ProFTPD server:
$ sudo /etc/init.d/proftpd start
or $ sudo service proftpd restart
Make sure that the default FTP port 21 is opened on the server:
$ sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
$ sudo iptables-save
Start Up ProFTPD automatically on server boot
$ sudo update-rc.d proftpd defaults


Configure /etc/proftpd/proftpd.conf:

#********************************************************************************************************************************************************************************************************************************************************************************************************************************************************************
#********************************************************************************************************************************************************************************************************************************************************************************************************************************************************************

Includes DSO modules
Include /etc/proftpd/modules.conf

# Basics, some site-specific
ServerName                      "Public Galaxy FTP"
ServerType                      standalone
DefaultServer                   on
Port                            21
Umask                           022
SyslogFacility                 DAEMON
SyslogLevel                    debug
MaxInstances                    30
User                            galaxy
Group                           galaxy

# Passive port range for the firewall
PassivePorts                    30000 40000



# Cause every FTP user to be "jailed" (chrooted) into their home directory
DefaultRoot                     ~

# Automatically create home directory if it doesn't exist
CreateHome                      on dirmode 700

# Allow users to overwrite their files
AllowOverwrite                  on

# Allow users to resume interrupted uploads
AllowStoreRestart               on

# Bar use of SITE CHMOD
<Limit SITE_CHMOD>
    DenyAll
</Limit>

# Bar use of RETR (download) since this is not a public file drop
<Limit RETR>
    DenyAll
</Limit>

# Do not authenticate against real (system) users
AuthPAM                         off

# Set up mod_sql_password - Galaxy passwords are stored as hex-encoded SHA1
SQLPasswordEngine               on
SQLPasswordEncoding             base64
SQLPasswordPBKDF2               SHA256 10000 24
SQLPasswordUserSalt             sql:/GetUserSalt

# Set up mod_sql to authenticate against the Galaxy database
SQLEngine                       on
SQLBackend                      postgres
SQLConnectInfo                  galaxy@/var/run/postgresql galaxy 1234
SQLAuthTypes                    PBKDF2
SQLAuthenticate                 users

# An empty directory in case chroot fails
SQLDefaultHomedir               /var/opt/local/proftpd

# Define a custom query for lookup that returns a passwd-like entry. Replace 512s with the UID and GID of the user running the Galaxy server
SQLUserInfo                     custom:/LookupGalaxyUser
SQLNamedQuery                   LookupGalaxyUser SELECT "email, (CASE WHEN substring(password from 1 for 6) = 'PBKDF2' THEN substring(password from 38 for 69) ELSE password END) AS password2,1001,1001,'/home/galaxy/galaxy-dist/database/ftp/%U','/bin/bash' FROM galaxy_user WHERE email='%U'"
SQLNamedQuery                   GetUserSalt SELECT "(CASE WHEN SUBSTRING (password from 1 for 6) = 'PBKDF2' THEN SUBSTRING (password from 21 for 16) END) AS salt FROM galaxy_user WHERE email='%U'"

#********************************************************************************************************************************************************************************************************************************************************************************************************************************************************************
#********************************************************************************************************************************************************************************************************************************************************************************************************************************************************************

Configure /etc/proftpd/modules.conf, add :
#********************************************************************************************************************************************************************************************************************************************************************************************************************************************************************
LoadModule mod_sql.c
LoadModule mod_sql_passwd.c
LoadModule mod_sql_postgres.c
LoadModule mod_sftp_sql.c
#********************************************************************************************************************************************************************************************************************************************************************************************************************************************************************


#see docker filesystem:
#docker run --rm -it --entrypoint=/bin/bash bgruening/galaxy-stable

Scaling and Load Balancing
**************************


#ref: https://galaxyproject.org/admin/config/performance/scaling/
#ref2: https://galaxyproject.org/events/bio-it-world2014/w14/

#Enable multiple cores in the virtual environment
Stop the VM and go to Settings -> System -> Process 
#(https://www.virtualbox.org/manual/ch03.html)
Change the processors number and check "Enable PAE/NX", as "Some operating systems (such as Ubuntu Server) require PAE support from the CPU and cannot be run in a virtual machine without it." 

Set up uWSGI
=============

#In galaxy.ini, define one or more [server:...] sections:

[server:web0]
use = egg:Paste#http
port = 8080
host = 192.168.56.11
use_threadpool = True
threadpool_workers = 7

[server:web1]
use = egg:Paste#http
port = 8081
host = 192.168.56.11
use_threadpool = True
threadpool_workers = 7
#Two are shown, you should create as many as are suitable for your usage and hardware. On our eight-core server, I run six web server processes. You may find you only need one, which is a slightly simpler configuration.


#In galaxy.ini, define a [uwsgi] section:

[uwsgi]
processes = 4
stats = 192.168.56.11:9191
socket = 192.168.56.11:4001
pythonpath = lib
threads = 4
logto = /home/galaxy/gonramp/logs/uwsgi.log #anywhere you like
master = True

#Port numbers for stats and socket can be adjusted as desired. Moreover, in the [app:main] section, you must set:

static_enabled = False
track_jobs_in_database = True

# Install wusgi
# use pip install in the virtual environment 
$ source gonramp/bin/activate
$ pip install uwsgi

#The web processes can then be started under uWSGI using:
$ cd /path/to/galaxy-dist
$ PYTHONPATH=eggs/PasteDeploy-1.5.0-py2.7.egg uwsgi --ini-paste config/galaxy.ini
#Once started, a proxy server (typically Apache or nginx) must be configured to proxy requests to uWSGI (using uWSGI's native protocol). Configuration details for these can be found in Proxy section.

Job Handler(s)
===============

#In galaxy.ini, define one or more additional [server:...] sections:

[server:handler0]
use = egg:Paste#http
port = 8090
host = 192.168.56.11
use_threadpool = true
threadpool_workers = 5

[server:handler1]
use = egg:Paste#http
port = 8091
host = 192.168.56.11
use_threadpool = true
threadpool_workers = 5

Configure job_conf.xml 
=======================
# uncomment in galaxy.init
job_config_file = config/job_conf.xml
#job configuration https://galaxyproject.org/admin/config/jobs/
<?xml version="1.0"?>
<!-- A sample job config that explicitly configures job running the way it is configured by default (if there is no explicit config). -->
<job_conf>
    <plugins workers="2">
        <plugin id="gridengine" type="runner" load="galaxy.jobs.runners.drmaa:DRMAAJobRunner"/>
    </plugins>
    <handlers default="handlers">
        <handler id="handler0" tags="handlers"/>
        <handler id="handler1" tags="handlers"/>
    </handlers>
    <destinations default="gridengine">
        <destination id="gridengine" runner="gridengine"/>
    </destinations>
    <limits>
        <limit type="registered_user_concurrent_jobs">2</limit>
        <limit type="unregistered_user_concurrent_jobs">1</limit>
        <limit type="job_walltime">24:00:00</limit>
    </limits>
</job_conf>

Install and configure Grid Engine
=================================

$ sudo apt-get install gridengine-master gridengine-exec gridengine-client gridengine-drmaa1.0

#You'll be asked a series of configuration questions for Postfix and Grid Engine at this point. The following answers are suitable for this workshop:

General type of mail configuration: Local only
System mail name: galaxy
Configure SGE automatically?: Yes
SGE cell name: default
SGE master hostname: galaxy

Start and Stop with supervisord
===============================

#Since you need to run multiple processes, the typical run.sh method for starting and stopping Galaxy won't work. The current recommended way to manage these multiple processes is with Supervisord. 
# install supervisor in virtualenv
$ pip install supervisor
# Creating a Configuration File
echo_supervisord_conf > /etc/supervisord.conf
# Configure
$ vim /etc/supervisord.conf
# add program galaxy_uwsgi, handler and group section

[program:galaxy_uwsgi]
command = /home/galaxy/gonramp/bin/uwsgi --virtualenv /home/galaxy/galaxy-dist/.venv --ini-paste /home/galaxy/galaxy-dist/config/galaxy.ini
directory = /home/galaxy/galaxy-dist
umask = 022
autostart = true
autorestart = true
startsecs = 20
user = galaxy
environment = PATH=/home/galaxy/galaxy-dist/.venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin, PYTHONHOME=/home/galaxy/galaxy-dist/.venv
numprocs = 1
stopsignal = INT
startretries = 15

[program:handler]
command = /home/galaxy/galaxy-dist/.venv/bin/python ./lib/galaxy/main.py -c ./config/galaxy.ini --server-name=handler%(process_num)s --log-file=/home/galaxy/galaxy-dist/handler%(process_num)s.log
directory = /home/galaxy/galaxy-dist
process_name = handler%(process_num)s
numprocs = 2
unmask = 022
autostart = true
autorestart = true
startsecs = 20
user = galaxy
environment = PYTHONHOME=/home/galaxy/galaxy-dist/.venv
;PYTHON_EGG_CACHE = /home/galaxy/galaxy-dist/.python-eggs, DRMAA_LIBRARY_PATH=/usr/lib/gridengine-drmaa/lib/libdrmaa.so.1.0, SGE_ROOT=/var/lib/gridengine

[group:galaxy]
programs = handler, galaxy_uwsgi

# Start the program
$ supervisorctl start galaxy:*
# Check status
$ supervisorctl status
# Log file
$ tail -f /tmp/supervisord.log

Proxy uWSGI with nginx
======================

# add server block below in /etc/nginx/nginx.conf and comment 
#include /etc/nginx/sites-enabled/*;
server {
    listen 80;
    root /var/www/html;
    # maximum file upload size
    client_max_body_size 10G;
    uwsgi_read_timeout 180;
    location /gonramp {
        include uwsgi_params;
        uwsgi_pass 192.168.56.11:4001;
        uwsgi_param UWSGI_SCHEME $scheme;
    }
    # directly serve static content in nginx
    location /gonramp/static {
        alias /home/galaxy/galaxy-dist/static;
        expires 24h;
    }
    location /gonramp/static/style {
        alias /home/galaxy/galaxy-dist/static/style/blue;
        expires 24h;
    }
    location /gonramp/static/scripts {
        alias /home/galaxy/galaxy-dist/static/scripts;
        expires 24h;
    }
    location /gonramp/favicon.ico {
        alias /home/galaxy/galaxy-dist/static/favicon.ico;
        expires 24h;
    }
    location /gonramp/robots.txt {
        alias /home/galaxy/galaxy-dist/static/robots.txt;
        expires 24h;
    }
}

# Restart nginx
$ server restart nginx

Access to gonramp
=================

$ supervisorctl start galaxy:*
#Goto http://192.168.56.11/gonramp






  

  
  


   

    

  
  
  


