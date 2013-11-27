=============
TZS-APPLIANCE
=============

Overview
========
    Tizen Simple Appliance (TZS) is a preconfigured build infrastructure for Tizen projects. It contains Gerrit (version control and review system), OBS (build system) and Jenkins (continuous integration system) fully configured to support simple distribution build workflow. This document describes initial configuration of the appliance.
    
Installation of software packages
=================================

1. Install required packages

::
    zypper ar http://pkg.jenkins-ci.org/opensuse-stable/ jenkins
    zypper ar http://download.opensuse.org/repositories/OBS:/Server:/2.4/openSUSE_12.3/ obs
    zypper ar -G http://download.tizen.org/tools/latest-release/openSUSE_12.3/ tools
    zypper ar -G http://download.tizen.org/services/archive/13.08/openSUSE_12.3/ services
    zypper ref
    zypper in apache memcached postfix rsync mysql redis jenkins obs-api obs-server obs-source_service obs-service-gbs obs-worker rpm-tizen build-initvm-i586 jenkins-plugins jenkins-jobs-common obs-event-plugin
    zypper clean

2. Create redis conf file:

::
    cp /etc/redis/default.conf.example /etc/redis/redis.conf
    chmod o+r /etc/redis/redis.conf

3. Enable automatic start on boot

::

    for item in apache2 memcached postfix rsyncd mysql redis jenkins obsapidelayed obsdispatcher obspublisher obsrepserver obsscheduler obsservice obssrcserver obsworker; do echo $item; systemctl enable $item.service ; done

4. Check if all services have been enabled

::
    chkconfig --list | grep 'obs\|mysql\|redis\|jenkins'
    systemctl list-unit-files --type=service | grep 'apache2\|memcached\|postfix\|rsyncd'

5. Start basic services if needed:

::
    for item in memcached postfix rsync mysql redis ; do echo $item; systemctl start $item ; done

Gerrit configuration
====================

1. Get git repositories, related to Tizen IVI using repo:

::
    mkdir -p Tizen_IVI/manifest
    cd Tizen_IVI/manifest
    git init .
    wget http://download.tizen.org/releases/daily/tizen/ivi/ivi-release/tizen_20131004.4/builddata/manifest/tizen_20131004.4_ia32.xml -O manifest.xml
    git add manifest.xml
    git commmit -m Init
    cd ..
    repo init -u ./manifest -b master -m manifest.xml --mirror
    repo sync

2. Tag them for future reference

::
    repo forall -c 'git tag -f release/tizen_20131004.4 $(cut -f2 -d" " HEAD)'

3. Create gerrit user

::
    mkdir /srv/gerrit
    useradd gerrit -d /srv/gerrit
    chown -R gerrit.users /srv/gerrit

4. Install and configure gerrit according to `Gerrit installation guide <http://gerrit-documentation.googlecode.com/svn/Documentation/2.7/install.html>`_

::
    su - gerrit
    wget http://gerrit-releases.storage.googleapis.com/gerrit-2.7.war
    java -jar gerrit-2.7.war init -d /srv/gerrit/review_site

    cat review_site/etc/gerrit.config 
    [gerrit]
    basePath = git
    canonicalWebUrl = https://<fqdn>/review/
    [database]
    type = mysql
    hostname = localhost
    database = reviewdb
    username = gerrit
    [auth]
    type = HTTP
    [sendemail]
    smtpServer = localhost
    [container]
    user = gerrit
    javaHome = /usr/lib64/jvm/java-1.7.0-openjdk-1.7.0/jre
    [sshd]
    listenAddress = *:29418
    [httpd]
    listenUrl = proxy-https://127.0.0.1:8081/review/
    [cache]
    directory = cache

5. Run gerrit
6. Log in as Admin user. First user, created in Gerrit is given admin rights automatically.
7. Create jenkins user and include it into Maintainers group.

8. Put repos, synced in step 1 to /srv/gerrit/review_site/git. Restart Gerrit to pick them up.

9. Clone all repos:

::
    ssh jenkins@tzs.fi.intel.com gerrit ls-projects |while read prj ; do dir=$(dirname $prj); [ -d $dir ] && (pushd $dir; git clone jenkins@tzs.fi.intel.com:/$prj; popd);done

10. Create submit tags and push them for the build:

::
    ssh jenkins@tzs.fi.intel.com gerrit ls-projects |grep ^platform |while read prj ; do echo $prj; git --git-dir=$prj/.git tag submit/tizen/20131007.000000 release/tizen_20131004.4 ;git --git-dir=$prj/.git push origin submit/tizen/20131007.000000;done

OBS configuration
=================

1. Setup OBS according to /usr/share/doc/packages/obs-server/README.SETUP
    *set CN of ssl certificate and ServerName in /etc/apache2/vhosts.d/obs to the same name(tzs)*
    *set my $hostname = 'localhost'; in /usr/lib/obs/server/BSConfig.pm if hostname is not resolvable*
2. Login as Admin with default password 'opensuse' to WebUI
3. Change Admin password
4. Create user jenkins
5. Create project Targets:Tizen:IVI in web ui
6. Copy rpms from /srv/build/Tizen:IVI/standard/i586 on tizen.org to /srv/build/Targets:Tizen:IVI/standard/i586/:full and run

::
    obs_admin --rescan-repository Targets:Tizen:IVI standard i586
7. Create project Tizen:IVI, copy meta and project config from tizen.org
8. Add user 'jenkins' as a maintainer to Tizen:IVI project
9. Add Targets:Tizen:IVI repository to the configuration:

::
    <project name="Tizen:IVI">
    <title>Tizen:IVI</title>
    <description>Tizen IVI project</description>
    <person userid="Admin" role="maintainer"/>
    <person userid="Admin" role="bugowner"/>
    <person userid="robot" role="maintainer"/>
    <debuginfo>
    <enable/>
    <enable arch="i586"/>
    </debuginfo>
    <repository name="standard" rebuild="direct">
    <path repository="standard" project="Targets:Tizen:IVI"/>
    <arch>i586</arch>
    </repository>
    </project>


Jenkins configuration
=====================

1. Move jenkins home to /srv:

::
    # mv /var/lib/jenkins /srv/
    # ln -s /srv/jenkins /var/lib/jenkins
2. Generate ssh keypair for jenkins user:

::
    # su - jenkins -s /bin/bash
    jenkins@tzs:~> ssh-keygen -b 2048
3. Add jenkins user to /etc/apache2/passwd to be able to log in to Gerrit

::
    htpasswd2 -csb passwd jenkins <password>
4. Login to Gerrit UI as jenkins and add ssh public key
5. Login to Gerrit UI as Admin and add jenkins user to 'Non-Interactive Users' group. This will let Jenkins Gerrit Trigger plugin rights to monitor gerrit events.
6. Check if ssh connection works:

::
    jenkins@tzs:~> ssh localhost -p 29418 gerrit --help
7. Configure Gerrit Trigger plugin in Jenkins UI:

::
    Hostname: localhost
    Frontend URL: http://localhost:8081
    Username: jenkins
8. Add jenkins user to kvm group to be able to run mic-appliance:

::
    usermod -a -G kvm jenkins


Apache configuration 
====================

Apache plays the following roles in the appliance:
* Serving static content: home page and download area
* Serving OBS webui and api web applications
* Redirection proxy to Jenkins and Gerrit
* http->https redirection
* Authentication of Jenkins and Gerrit users through basic http auth

1. Configuration is kept in /etc/apache2/vhosts.d/tzs.conf
2. Adding user to passwd file:

::
    htpasswd2 -csb /etc/apache/passwd <username> <password>
3. Example of working configuration:

::
    Listen 80
    Listen 444
    Listen 443

    # Passenger defaults
    PassengerSpawnMethod "smart" 
    PassengerMaxPoolSize 20
    #RailsEnv "development" 

    # allow long request urls and being part of headers
    LimitRequestLine 20000
    LimitRequestFieldsize 20000

    # Just the overview page
    <VirtualHost *:80>
            # TZS homepage
            DocumentRoot  "/srv/www/tzs" 

            <Directory /srv/www/tzs>
            Options Indexes FollowSymLinks
            Allow from all
            </Directory>

            # Redirections http->https      
            RewriteEngine On
            RewriteCond %{HTTPS} off
            RewriteRule ^/review/.*$ https://%{HTTP_HOST}%{REQUEST_URI}
            RewriteRule ^/ci/.*$ https://%{HTTP_HOST}%{REQUEST_URI}
            RewriteRule ^/build/$ https://%{HTTP_HOST}
    </VirtualHost>
    # OBS API
    <VirtualHost *:444>
            ServerName tzs

            #  General setup for the virtual host
            DocumentRoot  "/srv/www/obs/api/public" 
            ErrorLog /srv/www/obs/api/log/apache_error_log
            TransferLog /srv/www/obs/api/log/apache_access_log

            PassengerMinInstances 2
            PassengerPreStart https://api:444

            SSLEngine on

            #  SSL protocols
            #  Supporting TLS only is adequate nowadays
            SSLProtocol all -SSLv2 -SSLv3

            #   SSL Cipher Suite:
            #   List the ciphers that the client is permitted to negotiate.
            #   We disable weak ciphers by default.
            #   See the mod_ssl documentation or "openssl ciphers -v" for a
            #   complete list.
            SSLCipherSuite ALL:!aNULL:!eNULL:!SSLv2:!LOW:!EXP:!MD5:@STRENGTH

            SSLCertificateFile /srv/obs/certs/server.crt
            SSLCertificateKeyFile /srv/obs/certs/server.key

            <Directory /srv/www/obs/api/public>
            AllowOverride all
            Options -MultiViews

            # This requires mod_xforward loaded in apache 
            # Enable the usage via options.yml
            # This will decrease the load due to long running requests a lot (unloading from rails stack)
            XForward on

            Allow from all
            </Directory>                                                                                                                                                                                         

            SetEnvIf User-Agent ".*MSIE [1-5].*" \
            nokeepalive ssl-unclean-shutdown \
            downgrade-1.0 force-response-1.0

            CustomLog /var/log/apache2/ssl_request_log   ssl_combined

    </VirtualHost>
    # OBS WEB interface
    <VirtualHost *:443>
            ServerName tzs

            #  General setup for the virtual host
            DocumentRoot  "/srv/www/obs/webui/public" 
            ErrorLog /srv/www/obs/webui/log/apache_error_log
            TransferLog /srv/www/obs/webui/log/apache_access_log

            PassengerPreStart https://build

            SSLEngine on
            SSLProxyEngine on # required for raw buildlog

            #  SSL protocols
            #  Supporting TLS only is adequate nowadays
            SSLProtocol all -SSLv2 -SSLv3

            #   SSL Cipher Suite:
            #   List the ciphers that the client is permitted to negotiate.
            #   We disable weak ciphers by default.
            #   See the mod_ssl documentation or "openssl ciphers -v" for a
            #   complete list.
            SSLCipherSuite ALL:!aNULL:!eNULL:!SSLv2:!LOW:!EXP:!MD5:@STRENGTH

            SSLCertificateFile /srv/obs/certs/server.crt
            SSLCertificateKeyFile /srv/obs/certs/server.key

            <Directory /srv/www/obs/webui/public>
            AllowOverride all
            Options -MultiViews

            # This requires mod_xforward loaded in apache 
            # Enable the usage via options.yml
            # This will decrease the load due to long running requests a lot (unloading from rails stack)
            XForward on

            Allow from all
            </Directory>                                                                                                                                                                                         
            # from http://guides.rubyonrails.org/asset_pipeline.html
            <LocationMatch "^/assets/.*$">
            Header unset ETag
            FileETag None
            # RFC says only cache for 1 year
            ExpiresActive On
            ExpiresDefault "access plus 1 year" 
            </LocationMatch>
            # --- Proxy redirects for Gerrit and Jenkins --------
            ProxyRequests Off
            ProxyVia Off
            ProxyPreserveHost On

            <Proxy *>
            Order deny,allow
            Allow from all
            </Proxy>

            ProxyPass /review/ http://localhost:8081/review/

            ProxyPass         /ci/  http://localhost:8082/ci/
            ProxyPass         /ci  http://localhost:8082/ci
            ProxyPassReverse  /ci/  http://localhost:8082/ci/
            ProxyPassReverse  /ci  http://localhost:8082/ci
            ProxyPassReverse  /ci/  https://tzf.fi.intel.com/ci/
            ProxyPassReverse  /ci  https://tzf.fi.intel.com/ci

            <Location /review/login/>
            AuthUserFile  /etc/apache2/passwd 
            AuthType Basic
            AuthName "Gerrit Code Review" 
            Require valid-user
            </Location>

            <Location /ci/>
            AuthType Basic
            AuthName "Jenkins" 
            AuthUserFile "/etc/apache2/passwd" 
            Require valid-user
            </Location>
            # ------------------------------

            SetEnvIf User-Agent ".*MSIE [1-5].*" \
            nokeepalive ssl-unclean-shutdown \
            downgrade-1.0 force-response-1.0
            ## Older firefox versions needs this, otherwise it wont cache anything over SSL.
            Header append Cache-Control "public" 

            CustomLog /var/log/apache2/ssl_request_log   ssl_combined
    </VirtualHost>

Submitting all projects to OBS
==============================

This is done by tagging all projects with submit tag and pushing it to Gerrit:

::
    ls | while read dir; do echo $dir; git --git-dir $dir/.git tag -f submit/tizen/20130918.200000 origin/tizen; git --git-dir $dir/.git push -f origin submit/tizen/20130918.200000; done

Jenkins submit job should be triggered for every submit tag and upload tagged sources to OBS. After build is done <path repository="standard" project="Targets:Tizen:IVI"/> should be removed from target repository meta.
