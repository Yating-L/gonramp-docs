Create G-OnRamp Production Image
===================================

Requirements
------------

1. Download and install Virtual Box => https://www.virtualbox.org
2. Download Ubuntu Server 16.04.1 LTS => https://www.ubuntu.com/download/server

Step by step instruction
-------------------------
Install Ubuntu server to the VirtualBox
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

Initial Ubuntu setup
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

Set up internet host-only adapter (used to connect guest VM from host by ssh)
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

Install Galaxy
*****************
Running as an existing user will cause problems down the line when you want to grant or restrict access to data.

Create a NON-ROOT user called galaxy::

  $ sudo adduser galaxy
  password: 2016
  Add galaxy to sudo 
  $ usermod -aG sudo galaxy
  Then login with galaxy
  $ su - galaxy


Make sure Galaxy is using a clean Python interpreter. Conflicts in $PYTHONPATH or the interpreter's site-packages/ directory could cause problems. Galaxy manages its own dependencies for the framework, so you do not need to worry about these. 

The easiest way to do this is with a virtualenv::

  $ sudo apt-get update
  $ sudo apt-get install python-pip
  $ pip install --upgrade pip
  $ sudo pip install virtualenv
  $ virtualenv gonramp
  $ source gonramp/bin/activate
 
Galaxy requires a few things to run: a virtualenv, configuration files and dependent python modules. Starting the server at the first time will set these thing up. 

Download Galaxy 17.01 and rename galaxy folder::

  $ git clone -b release_17.01 https://github.com/galaxyproject/galaxy.git
  $ mv galaxy/ galaxy-dist

Basic configure Galaxy (galaxy.ini)::

  [server:main]
  # The address on which to listen.  By default, only listen to localhost (Galaxy
  # will not be accessible over the network).  Use '0.0.0.0' to listen on all
  # available network interfaces.
  host = 192.168.56.11
  debug = False
  use_interactive = False
  # filter-with = gzip
  cleanup_job = onsuccess


Set up PostgreSQL database
**************************

Install PostgreSQL::

  $ sudo apt-get update
  $ sudo apt-get install postgresql postgresql-contrib
  
Once installed, create a new database user and new database which the new user is the owner of. No further setup is required, since Galaxy manages its own schema. If you are using a UNIX socket to connect the application to the database (this is the standard case if Galaxy and the database are on the same system), you'll want to name the database user the same as the system user under which you run the Galaxy process.

Create a database and a new user for the database::

  $ sudo -u postgres createuser --superuser galaxy
  $ sudo -u galaxy createdb galaxy
  $ psql -U galaxy
  galaxy=# \password 
  Enter new password: 1234

In galaxy.ini, set::

  database_connection = postgresql://galaxy:1234@localhost/galaxy

Run Galaxy::

  $ cd galaxy
  $ sh run.sh

Set up proxy nginx
******************
`reference of how to install nginx <https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-16-04#step-3-check-your-web-server>`_

Install nginx
##############
Install nginx from pre-build package::

  $ sudo systemctl stop apache2.service
  $ sudo apt-get update
  $ sudo apt-get install nginx
  $ sudo apt-get install nginx-extras

..
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

Add galaxy server block
########################
add server block::

  $ cd /etc/nginx/sites-available/
  $ sudo vim galaxy

add following server block to galaxy file::

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
    location /gonramp/favicon.ico {
        alias /home/galaxy/galaxy-dist/static/favicon.ico;
        expires 24h;
    }
    location /gonramp/robots.txt {
        alias /home/galaxy/galaxy-dist/static/robots.txt;
        expires 24h;
    }
  }

enable galaxy server::

  $ cd /etc/nginx/sites-enabled/
  $ sudo ln -s /etc/nginx/sites-available/galaxy /etc/nginx/sites-enabled/galaxy
  
Make sure that you either comment out or modify line containing default configuration for enabled sites. in /etc/nginx/nginx.conf, include /etc/nginx/sites-enabled/::

  $ cd /etc/nginx/site-enabled
  $ rm default
  $ sudo service nginx restart
  
Galaxy application needs to be aware that it is running with a prefix (for generating URLs in dynamic pages). This is accomplished by configuring a Paste proxy-prefix filter in the [app:main] section of config/galaxy.ini and restarting Galaxy::
  
  [server:main]
    host = 192.168.56.11
  [filter:proxy-prefix]
    use = egg:PasteDeploy#prefix
    prefix = /gonramp
  [app:main]  
    filter-with = proxy-prefix
    cookie_path = /gonramp
    
.. 
    Change the root of nginx to /var/www/html/nginx, in order to distinct from apache page::
    
        $ cd /etc/nginx/site-available
        $ sudo vim galaxy #change root to /var/www/html/nginx
        $ cd /var/www/html
        $ sudo mkdir /var/www/html/nginx
        $ mv index.nginx-debian.html nginx/
        $ sudo systemctl restart nginx

Compression and caching
#######################
`nginx_ref <https://galaxyproject.org/admin/config/nginxProxy/>`_

All of Galaxy's static content can be cached on the client side, and everything (including dynamic content) can be compressed on the fly. This will decrease download and page load times for your clients, as well as decrease server load and bandwidth usage. To enable, you'll need nginx gzip support (which is standard unless compiled with --without-http_gzip_module), and the following in your nginx.conf::

  #!highlight nginx
  http {
        gzip on;
        gzip_disable "msie6";

        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 4;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
  }

For caching, you'll need to add an expires directive to the location /static { } blocks (already added, see server block)

Sending files using nginx
#########################

Add following to galaxy server block::

  server {
        location gonramp/_x_accel_redirect/ {
            internal;
            alias /;
        }
    }
    
And the following to the [app:main] section of config/galaxy.ini::

  nginx_x_accel_redirect_base = /_x_accel_redirect

Receiving files using nginx
############################
To enable it, you must first download, compile and install nginx_upload_module. This means recompiling nginx.
TODO: need to install nginx from the source and recompile

Rotate log files
****************
To use logrotate to rotate Galaxy log files, add a new file named "galaxy" to /etc/logrotate.d/ directory with something like::

  PATH_TO_GALAXY_LOG_FILES {
    weekly
    rotate 8
    copytruncate
    compress
    missingok
    notifempty
  }

FTP server 
**********
`Reference Enabling upload to Galaxy via FTP <https://galaxyproject.org/admin/config/upload-via-ftp/>_`

in the config file, galaxy.ini, set::

  ftp_upload_dir = /home/galaxy/galaxy-dist/database/ftp/
  ftp_upload_site = 192.168.56.11

.. 
    Allow your FTP server to read Galaxy's database 
    ################################################
    You'll need to grant a user access to read emails and passwords from the Galaxy database. Although the user Galaxy connects with could be used, I prefer to use a least-privilege setup wherein a separate user is created for the FTP server which has permission to SELECT from the galaxy_user table and nothing else. In postgres this is accomplished with:
    (TODO: I used user galaxy instead of galaxyftp. Try to figure out why galaxyftp won't work)
    postgres@dbserver% createuser -SDR galaxyftp
    postgres@dbserver% psql galaxydb::

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
#################
`FTP server <https://kyup.com/tutorials/install-setup-ftp-server-proftpd/>`_

Install ProFTPD::

  $ sudo apt-get install proftpd
  $ sudo apt-get install proftpd-mod-pgsql
  $ sudo nano /etc/proftpd/proftpd.conf  
  
Configure /etc/proftpd/proftpd.conf::

  # Includes DSO modules
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

  # By default, Galaxy stores passwords using PBKDF2. 
  # Configuration that handles PBKDF2 encryption
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

  # Define a custom query for lookup that returns a passwd-like entry. Replace 1001s with the UID and GID of the user running the Galaxy server (to find out: $ id galaxy)
  SQLUserInfo                     custom:/LookupGalaxyUser
  SQLNamedQuery                   LookupGalaxyUser SELECT "email, (CASE WHEN substring(password from 1 for 6) = 'PBKDF2' THEN substring(password from 38 for 69) ELSE password END) AS password2,1001,1001,'/home/galaxy/galaxy-dist/database/ftp/%U','/bin/bash' FROM galaxy_user WHERE email='%U'"
  SQLNamedQuery                   GetUserSalt SELECT "(CASE WHEN SUBSTRING (password from 1 for 6) = 'PBKDF2' THEN SUBSTRING (password from 21 for 16) END) AS salt FROM galaxy_user WHERE email='%U'"

Configure /etc/proftpd/modules.conf, add::

  LoadModule mod_sql.c
  LoadModule mod_sql_passwd.c
  LoadModule mod_sql_postgres.c
  LoadModule mod_sftp_sql.c


When we are ready with the configuration we can start up the ProFTPD server::

  $ sudo /etc/init.d/proftpd start
  or $ sudo service proftpd restart
  
.. 
  Make sure that the default FTP port 21 is opened on the server::
  $ sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
  $ sudo iptables-save
  # Start Up ProFTPD automatically on server boot
  $ sudo update-rc.d proftpd defaults


Scaling and Load Balancing
**************************
`reference1 <https://galaxyproject.org/admin/config/performance/scaling/>`_

`reference2 <https://galaxyproject.org/events/bio-it-world2014/w14/>`_

Enable multiple cores in the virtual environment

#. Stop the VM and go to Settings -> System -> Process 
#. Change the processors number. I changed to 4.
#. Check "Enable PAE/NX", as "Some operating systems (such as Ubuntu Server) require PAE support from the CPU and cannot be run in a virtual machine without it." ( `reference <https://www.virtualbox.org/manual/ch03.html>`_)

Set up uWSGI
############
In galaxy.ini, define one or more [server:...] sections::

  #Two are shown, you should create as many as are suitable for your usage and hardware. 
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

In galaxy.ini, define a [uwsgi] section::

  [uwsgi]
  processes = 4
  stats = 192.168.56.11:9191
  socket = 192.168.56.11:4001
  pythonpath = lib
  threads = 4
  logto = /home/galaxy/gonramp/logs/uwsgi.log #anywhere you like
  master = True

Port numbers for stats and socket can be adjusted as desired. Moreover, in the [app:main] section, you must set::

  static_enabled = False
  track_jobs_in_database = True

Install wusgi::

  # use pip install in the virtual environment 
  $ source gonramp/bin/activate
  $ pip install uwsgi

The web processes can then be started under uWSGI using::

  $ cd /path/to/galaxy-dist
  $ PYTHONPATH=eggs/PasteDeploy-1.5.0-py2.7.egg uwsgi --ini-paste config/galaxy.ini
  #Once started, a proxy server (typically Apache or nginx) must be configured to proxy requests to uWSGI (using uWSGI's native protocol). Configuration details for these can be found in Proxy section.

Job Handler(s)
##############
In galaxy.ini, define one or more additional [server:...] sections::

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
######################
`Configure job reference <https://galaxyproject.org/admin/config/jobs/>`_

Uncomment in galaxy.ini::

  job_config_file = config/job_conf.xml

Configure galaxy-dist/conf/job_conf.xml::

  <?xml version="1.0"?>
  <!-- A sample job config that explicitly configures job running the way it is configured by default (if there is no explicit config). -->
  <job_conf>
    <plugins>
        <plugin id="local" type="runner" load="galaxy.jobs.runners.local:LocalJobRunner" workers="4"/>
    </plugins>
    <handlers default="handlers">
        <handler id="handler0" tags="handlers"/>
        <handler id="handler1" tags="handlers"/>
    </handlers>
    <destinations>
        <destination id="local" runner="local"/>
    </destinations>
  </job_conf>
  
..  
  # Sample for using gridengine
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

..
  TODO: Install and configure Grid Engine
  (Maybe not need for localsystem)
  #######################################
  Install Grid engine::

  $ sudo apt-get install gridengine-master gridengine-exec gridengine-client gridengine-drmaa1.0
  #You'll be asked a series of configuration questions for Postfix and Grid Engine at this point. The following answers are suitable for this workshop:
  General type of mail configuration: Local only
  System mail name: galaxy
  Configure SGE automatically?: Yes
  SGE cell name: default
  SGE master hostname: galaxy

Start and Stop with supervisord
###############################
Since you need to run multiple processes, the typical run.sh method for starting and stopping Galaxy won't work. The current recommended way to manage these multiple processes is with Supervisord. 

Install supervisor in virtualenv::

  $ pip install supervisor
  
Creating a Configuration File::

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

  [group:galaxy]
  programs = handler, galaxy_uwsgi

Start the program::

  $ supervisorctl start galaxy:*
  # Check status
  $ supervisorctl status
  # Log file
  $ tail -f /tmp/supervisord.log

Proxy uWSGI with nginx
######################
Add server block below in /etc/nginx/nginx.conf and comment out #include /etc/nginx/sites-enabled/ ::

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

Restart nginx::

  $ sudo service nginx restart

Access to gonramp
#################

Start then access through browser::
  
  $ supervisord
  # Check status
  $ supervisorctl status 
  # Restart after changing configuration
  $ supervisorctl restart galaxy:*  
  # Stop Galaxy
  $ supervisorctl stop galaxy:* 
  Goto http://192.168.56.11/gonramp


6. Set up G-OnRamp
******************

Become an Admin
###############
In order to install tools, you have to become administrator for your Galaxy instance. First start the server, go to http://192.168.56.11:8080/gonramp, and register as a new user with your email address::

  username: galaxyadmin@gonramp.org
  public name: galaxyadmin
  password: 12341234
  
Go to galaxy folder and find a sub-folder called config. Add a new file named galaxy.ini in the config folder. You can copy the content of galaxy.ini.sample into galaxy.ini. In galaxy.ini, search for the line containing “admin_users”. Add your user email address to admin users. (Replace None to your email address). You can add multiple admin users by appending another email and separating them with a comma::
  
  # this should be a comma-separated list of valid Galaxy users
  admin_users = galaxyadmin@gonramp.org
  
  
Set up conda
############

In galaxy.ini, uncomment and edit the following conda configuration::

  conda_ensure_channels = conda-forge,r,bioconda,iuc
  conda_auto_install = True
  conda_auto_init = True

..  
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

..  
    Add necessary datatypes
    #######################

    Copy the “datatypes_conf.xml.sample” and rename it to “datatypes_conf.xml”.
    Add the line below in between <registration></registration>::

  <datatype extension="psl" subclass="True" type="galaxy.datatypes.tabular:Tabular" />

.. 
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
- augustus 
- hisat2
- stringtie 
- blastXmlToPsl (by yating-l)
- trfbig (by yating-l)
- pslToBed 
- bamtobigwig (by yating-l)
- hubarchivecreator 
- multi_fasta_glimmer_hmm (by yating-l)
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

You need to use a trained organism by adding them as reference data in Galaxy. Add the glimmer_hmm_trained_dir data table to tool_data_table_conf.xml in $GALAXY_ROOT/config/::

  <!-- glimmer_hmm trained_dir -->
  <table name="glimmer_hmm_trained_dir" comment_char="#">
    <columns>value, name, path</columns>
    <file path="tool-data/glimmer_hmm.loc" />
  </table> 
  
Configure the glimmer_hmm.loc file referencing your trained organism, in tool-data. Uncomment the species and add the path to trained_dir, for example::
    
    #TAB separated
    human   Human   /home/galaxy/Dependencies/GlimmerHMM/trained_dir/human
    celegans        Celegan /home/galaxy/Dependencies/GlimmerHMM/trained_dir/celegans
    arabidopsis     Arabidopsis     /home/galaxy/Dependencies/GlimmerHMM/trained_dir/arabidopsis
    rice    Rice    /home/galaxy/Dependencies/GlimmerHMM/trained_dir/rice
    zebrafish       Zebrafish       /home/galaxy/Dependencies/GlimmerHMM/trained_dir/zebrafish

2. jbrowsearchivecreator

Install JBrowse-1.12.1 at /var/www/html::

    $ cd /var/www/html
    $ wget --trust-server-names http://jbrowse.org/wordpress/wp-content/plugins/download-monitor/download.php?id=105
    $ sudo apt-get install unzip
    $ sudo unzip JBrowse-1.12.1.zip 
    $ cd JBrowse-1.12.1
    $ sudo ./setup.sh
    # add a subdir to store hub data
    $ sudo mkdir data
    $ sudo chown -R galaxy:galaxy data

Add G-OnRamp plugins::

    $ cd JBrowse-1.12.1/plugins
    $ sudo git clone https://github.com/Yating-L/JBrowse_plugins.git G-OnRamp_plugin

Add a plugins configuration variable in your jbrowse_conf.json file in the top-level JBrowse directory, and add an entry telling JBrowse where the plugin is. Example::

    {
     "plugins": [ 'G-OnRamp_plugin' ]

    }
 





  

  
  


   

    

  
  
  


