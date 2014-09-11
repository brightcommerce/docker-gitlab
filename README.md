# Docker GitLab

Dockerfile to build a GitLab container image. We've been somewhat opinionated with regard to the GitLab options available, for instance we use PostgreSQL not MySQL, we don't install Redis Server locally and we don't integrate with external issue trackers such as Redmine or Jira. MySQL client drivers are still installed to satisfy the gem requirements of the GitLab Rails application.

## Table of Contents
- [Version](#version)
- [Hardware Requirements](#hardware-requirements)
    - [CPU](#cpu)
    - [Memory](#memory)
    - [Storage](#storage)
- [Supported Web Browsers](#supported-web-browsers)
- [Installation](#installation)
- [How To Use](#how-to-use)
- [Configuration](#configuration)
    - [Ports](#ports)
    - [Data Store](#data-store)
        - [Database](#database)
          - [External PostgreSQL Server](#external-postgresql-server)
          - [Linking to PostgreSQL Container](#linking-to-postgresql-container)
        - [Redis](#redis)
          - [External Redis Server](#external-redis-server)
          - [Linking to Redis Container](#linking-to-redis-container)
    - [Mail](#mail)
    - [SSL](#ssl)
      - [Generation of Self Signed Certificates](#generation-of-self-signed-certificates)
      - [Strengthening Server Security](#strengthening-the-server-security)
      - [Installation of the Certificates](#installation-of-the-certificates)
      - [Enabling HTTPS support](#enabling-https-support)
      - [Using HTTPS with a load balancer](#using-https-with-a-load-balancer)
      - [Establishing trust with your server](#establishing-trust-with-your-server)
      - [Installing Trusted SSL Server Certificates](#installing-trusted-ssl-server-certificates)
    - [Deploy to a subdirectory (relative url root)](#deploy-to-a-subdirectory-relative-url-root)
    - [Putting it all together](#putting-it-all-together)
    - [OmniAuth Integration](#omniauth-integration)
      - [Google](#google)
      - [Twitter](#twitter)
      - [GitHub](#github)
    - [Available Configuration Parameters](#available-configuration-parameters)
- [Maintenance](#maintenance)
    - [Creating Backups](#creating-backups)
    - [Restoring Backups](#restoring-backups)
    - [Automated Backups](#automated-backups)
    - [Shell Access](#shell-access)
- [Upgrading](#upgrading)
- [Rake Tasks](#rake-tasks)
- [References](#references)
- [Credit](#credit)

## Version

The current release (7.2.1) contains scripts to install GitLab v7.2.1 and GitLab Shell v1.9.7, and uses the Brightcommerce Ubuntu 14.04 base image. Our version numbers will reflect the version of GitLab being installed.

## Hardware Requirements

### CPU

- 1 core works for under 100 users but the responsiveness might suffer
- 2 cores is the recommended number of cores and supports up to 100 users
- 4 cores supports up to 1,000 users
- 8 cores supports up to 10,000 users

### Memory

- 512MB is too little memory, GitLab will be very slow and you will need 250MB of swap
- 768MB is the minimal memory size but we advise against this
- 1GB supports up to 100 users (with individual repositories under 250MB, otherwise git memory usage necessitates using swap space)
- **2GB** is the **recommended** memory size and supports up to 1,000 users
- 4GB supports up to 10,000 users

### Storage

The necessary hard drive space largely depends on the size of the repos you want to store in GitLab. But as a *rule of thumb* you should have at least twice as much free space as your all repos combined take up. You need twice the storage because [GitLab satellites](https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/structure.md) contain an extra copy of each repo.

If you want to be flexible about growing your hard drive space in the future consider mounting it using LVM so you can add more hard drives when you need them.

Apart from a local hard drive you can also mount a volume that supports the network file system (NFS) protocol. This volume might be located on a file server, a network attached storage (NAS) device, a storage area network (SAN) or on an Amazon Web Services (AWS) Elastic Block Store (EBS) volume.

If you have enough RAM memory and a recent CPU the speed of GitLab is mainly limited by hard drive seek times. Having a fast drive (7200 RPM and up) or a solid state drive (SSD) will improve the responsiveness of GitLab.

## Supported Web Browsers

- Chrome (Latest stable version)
- Firefox (Latest released version)
- Safari 7+ (Known problem: required fields in HTML5 do not work)
- Opera (Latest released version)
- IE 10+

## Installation

Pull the latest version of the image from the Docker Index. This is the recommended method of installation as it is easier to update image in the future. These builds are performed by the **Docker Trusted Build** service.

``` bash
docker pull brightcommerce/gitlab:latest
```

or specify a tagged version:

``` bash
docker pull brightcommerce/gitlab:7.2.1
```

Alternately you can build the image yourself:

``` bash
git clone https://github.com/brightcommerce/docker-gitlab.git
cd docker-gitlab
docker build --tag="$USER/gitlab" .
```

## How To Use

Run the GitLab image:

``` bash
docker run --name='gitlab' -it --rm -p 10022:22 -p 10080:80 -e 'GITLAB_PORT=10080' -e 'GITLAB_SSH_PORT=10022' brightcommerce/gitlab:7.2.1
```

__NOTE__: Please allow a couple of minutes for the GitLab application to start.

Point your browser to `http://localhost:10080` and login using the default username and password:

* username: **root**
* password: **5iveL!fe**

You should now have the GitLab application up and ready for testing. If you want to use this image in production then please read on.

## Configuration

### Ports

This installation exposes ports `22` (ssh), `80` (http) and `443` (https).

### Data Store

GitLab is a code hosting software and as such you don't want to lose your code when the Docker container is stopped/deleted. To avoid losing any data, you should mount a volume at,

* `/home/git/data`

SELinux users are also required to change the security context of the mount point so that it plays nicely with SELinux.

``` bash
mkdir -p /opt/gitlab/data
sudo chcon -Rt svirt_sandbox_file_t /opt/gitlab/data
```

Volumes can be mounted in Docker by specifying the **'-v'** option in the `docker run` command.

``` bash
docker run --name=gitlab -d -v /opt/gitlab/data:/home/git/data brightcommerce/gitlab:7.2.1
```

#### Database

GitLab uses a database backend to store its' data. You can only configure this image to use PostgreSQL.

*Note: GitLab HQ recommends using PostgreSQL*

##### External PostgreSQL Server

The image supports using an external PostgreSQL Server. This is also controlled via environment variables.

``` sql
CREATE ROLE gitlab with LOGIN CREATEDB PASSWORD 'password';
CREATE DATABASE gitlabhq_production;
GRANT ALL PRIVILEGES ON DATABASE gitlabhq_production to gitlab;
```

To make sure the database is initialized start the container with `app:rake gitlab:setup` option.

*Assuming that the PostgreSQL server host is 192.168.1.100*

``` bash
docker run --name=gitlab -it --rm -e 'DB_TYPE=postgres' -e 'DB_HOST=192.168.1.100' -e 'DB_NAME=gitlabhq_production' -e 'DB_USER=gitlab' -e 'DB_PASS=password' -v /opt/gitlab/data:/home/git/data brightcommerce/gitlab:7.2.1 app:rake gitlab:setup
```

**NOTE: The above setup is performed only for the first run**.

This will initialize the GitLab database. Now that the database is initialized, start the container normally.

``` bash
docker run --name=gitlab -d -e 'DB_TYPE=postgres' -e 'DB_HOST=192.168.1.100' -e 'DB_NAME=gitlabhq_production' -e 'DB_USER=gitlab' -e 'DB_PASS=password' v /opt/gitlab/data:/home/git/data brightcommerce/gitlab:7.2.1
```

##### Linking to a PostgreSQL Container

You can link this image with a PostgreSQL container for the database requirements. The alias of the PostgreSQL server container should be set to **postgresql** while linking with the GitLab image.

If a PostgreSQL container is linked, only the `DB_TYPE`, `DB_HOST` and `DB_PORT` settings are automatically retrieved using the linkage. You may still need to set other database connection parameters such as the `DB_NAME`, `DB_USER`, `DB_PASS` and so on.

To illustrate linking with a PostgreSQL container, we will use the [brightcommerce/postgresql](https://github.com/brightcommerce/docker-postgresql) image. When using the PostgreSQL image in production you should mount a volume for the PostgreSQL data store. Please refer the [README](https://github.com/brightcommerce/docker-postgresql/blob/master/README.md) of docker-postgresql for details.

First, lets pull the PostgreSQL image from the Docker Index:

``` bash
docker pull brightcommerce/postgresql:latest
```

For data persistence lets create a store for the PostgreSQL and start the container.

The updated run command looks like this:

``` bash
docker run --name=postgresql -d -v /opt/postgresql/data:/var/lib/postgresql brightcommerce/postgresql:latest
```

You should now have the PostgreSQL server running. The password for the postgres user can be found in the logs of the PostgreSQL image.

``` bash
docker logs postgresql
```

Now, let's login to the PostgreSQL server and create a user and database for the GitLab application:

``` bash
docker run -it --rm brightcommerce/postgresql:latest psql -U postgres -h $(docker inspect --format {{.NetworkSettings.IPAddress}} postgresql)
```

``` sql
CREATE ROLE gitlab with LOGIN CREATEDB PASSWORD 'password';
CREATE DATABASE gitlabhq_production;
GRANT ALL PRIVILEGES ON DATABASE gitlabhq_production to gitlab;
```

Now that we have the database created for GitLab, let's install the database schema. This is done by starting the GitLab container with the `app:rake gitlab:setup` command:

``` bash
docker run --name=gitlab -it --rm --link postgresql:postgresql -e 'DB_USER=gitlab' -e 'DB_PASS=password' -e 'DB_NAME=gitlabhq_production' -v /opt/gitlab/data:/home/git/data brightcommerce/gitlab:7.2.1 app:rake gitlab:setup
```

**NOTE: The above setup is performed only for the first run**.

We are now ready to start the GitLab application.

``` bash
docker run --name=gitlab -d --link postgresql:postgresql -e 'DB_USER=gitlab' -e 'DB_PASS=password' -e 'DB_NAME=gitlabhq_production' -v /opt/gitlab/data:/home/git/data brightcommerce/gitlab:7.2.1
```

#### Redis

GitLab uses the Redis Server for its key-value data store. The Redis server connection details can be specified using environment variables.

##### External Redis Server

The image can be configured to use an external Redis server. The configuration should be specified using environment variables while starting the GitLab image.

*Assuming that the Redis server host is 192.168.1.100*

``` bash
docker run --name=gitlab -it --rm -e 'REDIS_HOST=192.168.1.100' -e 'REDIS_PORT=6379' brightcommerce/gitlab:7.2.1
```

##### Linking to a Redis Container

You can link this image with a Redis container to satisfy GitLab's Redis requirement. The alias of the Redis server container should be set to **redisio** while linking with the GitLab image.

To illustrate linking with a Redis container, we will use the [brightcommerce/redis](https://github.com/brightcommerce/docker-redis) image. Please refer the [README](https://github.com/brightcommerce/docker-redis/blob/master/README.md) of docker-redis for details.

First, let's pull the Redis image from the Docker Index:

``` bash
docker pull brightcommerce/redis:latest
```

Start the Redis container:

``` bash
docker run --name=redis -d brightcommerce/redis:latest
```

We are now ready to start the GitLab application:

``` bash
docker run --name=gitlab -d --link redis:redisio brightcommerce/gitlab:7.2.1
```

### Mail

The mail configuration should be specified using environment variables while starting the GitLab image. The configuration defaults to using Gmail to send emails and requires the specification of a valid username and password to login to the Gmail servers.

The following environment variables need to be specified to get email support to work:

* SMTP_ENABLED (defaults to `true` if `SMTP_USER` is defined, else defaults to `false`)
* SMTP_DOMAIN (defaults to `www.gmail.com`)
* SMTP_HOST (defaults to `smtp.gmail.com`)
* SMTP_PORT (defaults to `587`)
* SMTP_USER
* SMTP_PASS
* SMTP_STARTTLS (defaults to `true`)
* SMTP_AUTHENTICATION (defaults to `login` if `SMTP_USER` is set)

``` bash
docker run --name=gitlab -d -e 'SMTP_USER=USER@gmail.com' -e 'SMTP_PASS=PASSWORD' -v /opt/gitlab/data:/home/git/data brightcommerce/gitlab:7.2.1
```

### SSL

Access to the GitLab application can be secured using SSL so as to prevent unauthorized access to the data in your repositories. While a CA certified SSL certificate allows for verification of trust via the CA, self-signed certificates can also provide an equal level of trust verification as long as each client takes some additional steps to verify the identity of your website. I will provide instructions on achieving this towards the end of this section.

To secure your application via SSL you basically need two things:
- **Private key (.key)**
- **SSL certificate (.crt)**

When using CA certified certificates, these files are provided to you by the CA. When using self-signed certificates you need to generate these files yourself. Skip the following section if you are armed with CA certified SSL certificates.

Jump to the [Using HTTPS with a load balancer](#using-https-with-a-load-balancer) section if you are using a load balancer such as hipache, haproxy or nginx.

#### Generation of Self-Signed Certificates

Generation of self-signed SSL certificates involves a simple 3 step procedure.

**STEP 1**: Create the server private key:

``` bash
openssl genrsa -out gitlab.key 2048
```

**STEP 2**: Create the certificate signing request (CSR):

``` bash
openssl req -new -key gitlab.key -out gitlab.csr
```

**STEP 3**: Sign the certificate using the private key and CSR:

``` bash
openssl x509 -req -days 365 -in gitlab.csr -signkey gitlab.key -out gitlab.crt
```

Congratulations! you have now generated an SSL certificate that's valid for 365 days.

#### Strengthening Server Security

This section provides you with instructions to [strengthen your server security](https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html). To achieve this we need to generate stronger DHE parameters.

``` bash
openssl dhparam -out dhparam.pem 2048
```

#### Installation of the SSL Certificates

Out of the four files generated above, we need to install the `gitlab.key`, `gitlab.crt` and `dhparam.pem` files at the GitLab server. The CSR file is not needed, but do make sure you safely backup the file (in case you ever need it again).

The default path that the GitLab application is configured to look for the SSL certificates is at `/home/git/data/certs`, however this can be changed using the `SSL_KEY_PATH`, `SSL_CERTIFICATE_PATH` and `SSL_DHPARAM_PATH` configuration options.

If you remember from above, the `/home/git/data` path is the path of the [data store](#data-store), which means that we have to create a folder named certs inside `/opt/gitlab/data/` and copy the files into it and as a measure of security we will update the permission on the `gitlab.key` file to only be readable by the owner.

``` bash
mkdir -p /opt/gitlab/data/certs
cp gitlab.key /opt/gitlab/data/certs/
cp gitlab.crt /opt/gitlab/data/certs/
cp dhparam.pem /opt/gitlab/data/certs/
chmod 400 /opt/gitlab/data/certs/gitlab.key
```

Great! we are now just one step away from having our application secured.

#### Enabling HTTPS support

HTTPS support can be enabled by setting the `GITLAB_HTTPS` option to `true`. Additionally, when using self-signed SSL certificates you need to the set `SSL_SELF_SIGNED` option to `true` as well. Assuming we are using self-signed certificates

``` bash
docker run --name=gitlab -d -e 'GITLAB_HTTPS=true' -e 'SSL_SELF_SIGNED=true' -v /opt/gitlab/data:/home/git/data brightcommerce/gitlab:7.2.1
```

In this configuration, any requests made over the plain http protocol will automatically be redirected to use the https protocol. However, this is not optimal when using a load balancer.

#### Using HTTPS with a load balancer

Load balancers like nginx/haproxy/hipache talk to backend applications over plain http and as such the installation of ssl keys and certificates are not required and should **NOT** be installed in the container. The SSL configuration has to instead be done at the load balancer.

However, when using a load balancer you **MUST** set `GITLAB_HTTPS` to `true`. Additionally you will need to set the `SSL_SELF_SIGNED` option to `true` if self signed SSL certificates are in use.

With this in place, you should configure the load balancer to support handling of https requests. But that is out of the scope of this document. Please refer to [Using SSL/HTTPS with HAProxy](http://seanmcgary.com/posts/using-sslhttps-with-haproxy) for information on the subject.

When using a load balancer, you probably want to make sure the load balancer performs the automatic http to https redirection. Information on this can also be found in the link above.

In summation, when using a load balancer, the docker command would look for the most part something like this:

``` bash
docker run --name=gitlab -d -p 10022:22 -p 10080:80 -e 'GITLAB_SSH_PORT=10022' -e 'GITLAB_PORT=443' -e 'GITLAB_HTTPS=true' -e 'SSL_SELF_SIGNED=true' -v /opt/gitlab/data:/home/git/data brightcommerce/gitlab:7.2.1
```

Again, drop the `-e 'SSL_SELF_SIGNED=true'` option if you are using CA certified SSL certificates.

#### Establishing trust with your server

This section deals will self-signed ssl certificates. If you are using CA certified certificates, your done.

This section is more of a client-side configuration so as to add a level of confidence at the client to be 100 percent sure they are communicating with whom they think they are.

This is simply done by adding the servers certificate into their list of trusted certificates. On Ubuntu, this is done by copying the `gitlab.crt` file to `/usr/local/share/ca-certificates/` and executing `update-ca-certificates`.

Again, this is a client-side configuration which means that everyone who is going to communicate with the server should perform this configuration on their machine. In short, distribute the `gitlab.crt` file among your developers and ask them to add it to their list of trusted ssl certificates. Failure to do so will result in errors that look like this:

``` bash
git clone https://git.local.host/gitlab-ce.git
fatal: unable to access 'https://git.local.host/gitlab-ce.git': server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none
```

You can do the same at the web browser. Instructions for installing the root certificate for Firefox can be found [here](http://portal.threatpulse.com/docs/sol/Content/03Solutions/ManagePolicy/SSL/ssl_firefox_cert_ta.htm). You will find similar options for Chrome, just make sure you install the certificate under the authorities tab of the certificate manager dialog.

#### Installing Trusted SSL Server Certificates

If your GitLab CI server is using self-signed SSL certificates then you should make sure the GitLab CI server certificate is trusted on the GitLab server for them to be able to talk to each other.

The default path image is configured to look for the trusted SSL certificates is at `/home/git/data/certs/ca.crt`, this can however be changed using the `CA_CERTIFICATES_PATH` configuration option.

Copy the `ca.crt` file into the certs directory on the [datastore](#data-store). The `ca.crt` file should contain the root certificates of all the servers you want to trust. With respect to GitLab CI, this will be the contents of the gitlab_ci.crt file as described in the [README](https://github.com/brightcommerce/docker-gitlab-ci/blob/master/README.md#ssl) of the [docker-gitlab-ci](https://github.com/brightcommerce/docker-gitlab-ci) container.

By default, our own server certificate [gitlab.crt](#generation-of-self-signed-certificates) is added to the trusted certificates list.

### Deploy to a subdirectory (relative url root)

By default GitLab expects that your application is running at the root (eg. /). This section explains how to run your application inside a directory.

Let's assume we want to deploy our application to '/git'. GitLab needs to know this directory to generate the appropriate routes. This can be specified using the `GITLAB_RELATIVE_URL_ROOT` configuration option like so:

``` bash
docker run --name=gitlab -it --rm -e 'GITLAB_RELATIVE_URL_ROOT=/git' -v /opt/gitlab/data:/home/git/data  brightcommerce/gitlab:7.2.1
```

GitLab will now be accessible at the `/git` path, e.g. `http://www.example.com/git`.

**Note**: *The `GITLAB_RELATIVE_URL_ROOT` parameter should always begin with a slash and **SHOULD NOT** have any trailing slashes.*

### Putting it all together

``` bash
docker run --name=gitlab -d -h git.local.host -v /opt/gitlab/data:/home/git/data -v /opt/gitlab/mysql:/var/lib/mysql -e 'GITLAB_HOST=git.local.host' -e 'GITLAB_EMAIL=gitlab@local.host' -e 'SMTP_USER=USER@gmail.com' -e 'SMTP_PASS=PASSWORD' brightcommerce/gitlab:7.2.1
```

If you are using an external mysql database

``` bash
docker run --name=gitlab -d -h git.local.host -v /opt/gitlab/data:/home/git/data -e 'DB_HOST=192.168.1.100' -e 'DB_NAME=gitlabhq_production' -e 'DB_USER=gitlab' -e 'DB_PASS=password' \-e 'GITLAB_HOST=git.local.host' -e 'GITLAB_EMAIL=gitlab@local.host' -e 'SMTP_USER=USER@gmail.com' -e 'SMTP_PASS=PASSWORD' brightcommerce/gitlab:7.2.1
```

### OmniAuth Integration

GitLab leverages OmniAuth to allow users to sign in using Twitter, GitHub, and other popular services. Configuring OmniAuth does not prevent standard GitLab authentication or LDAP (if configured) from continuing to work. Users can choose to sign in using any of the configured mechanisms.

Refer to the GitLab [documentation](http://doc.gitlab.com/ce/integration/omniauth.html) for additional information.

#### Google

To enable the Google OAuth2 OmniAuth provider you must register your application with Google. Google will generate a client ID and secret key for you to use. Please refer to the GitLab [documentation](http://doc.gitlab.com/ce/integration/google.html) for the procedure to generate the client ID and secret key with google.

Once you have the client ID and secret keys generated, configure them using the `OAUTH_GOOGLE_API_KEY` and `OAUTH_GOOGLE_APP_SECRET` environment variables respectively.

For example, if your client ID is `xxx.apps.googleusercontent.com` and client secret key is `yyy`, then adding `-e 'OAUTH_GOOGLE_API_KEY=xxx.apps.googleusercontent.com' -e 'OAUTH_GOOGLE_APP_SECRET=yyy'` to the docker run command enables support for Google OAuth.

You can also restrict logins to a single domain by adding `-e 'OAUTH_GOOGLE_RESTRICT_DOMAIN=example.com'`. This is particularly useful when combined with `-e 'OAUTH_ALLOW_SSO=true'` and `-e 'OAUTH_BLOCK_AUTO_CREATED_USERS=false'`.

#### Twitter

To enable the Twitter OAuth2 OmniAuth provider you must register your application with Twitter. Twitter will generate a API key and secret for you to use. Please refer to the GitLab [documentation](http://doc.gitlab.com/ce/integration/twitter.html) for the procedure to generate the API key and secret with twitter.

Once you have the API key and secret generated, configure them using the `OAUTH_TWITTER_API_KEY` and `OAUTH_TWITTER_APP_SECRET` environment variables respectively.

For example, if your API key is `xxx` and the API secret key is `yyy`, then adding `-e 'OAUTH_TWITTER_API_KEY=xxx' -e 'OAUTH_TWITTER_APP_SECRET=yyy'` to the docker run command enables support for Twitter OAuth.

#### GitHub

To enable the GitHub OAuth2 OmniAuth provider you must register your application with GitHub. GitHub will generate a Client ID and secret for you to use. Please refer to the GitLab [documentation](http://doc.gitlab.com/ce/integration/github.html) for the procedure to generate the Client ID and secret with github.

Once you have the Client ID and secret generated, configure them using the `OAUTH_GITHUB_API_KEY` and `OAUTH_GITHUB_APP_SECRET` environment variables respectively.

For example, if your Client ID is `xxx` and the Client secret is `yyy`, then adding `-e 'OAUTH_GITHUB_API_KEY=xxx' -e 'OAUTH_GITHUB_APP_SECRET=yyy'` to the docker run command enables support for GitHub OAuth.

### Available Configuration Parameters

*Please refer the docker run command options for the `--env-file` flag where you can specify all required environment variables in a single file. This will save you from writing a potentially long docker run command.*

Below is the complete list of available options that can be used to customize your GitLab installation.

- **GITLAB_HOST**: The hostname of the GitLab server. Defaults to `localhost`.
- **GITLAB_PORT**: The port of the GitLab server. Defaults to `80` for plain http and `443` when https is enabled.
- **GITLAB_EMAIL**: The email address for the GitLab server.  Defaults to `example@example.com`.
- **GITLAB_SIGNUP**: Enable or disable user signups. Default is `false`.
- **GITLAB_SIGNIN**: If set to `false`, standard login form won't be shown on the sign-in page. Default is `true`.
- **GITLAB_PROJECTS_LIMIT**: Set default projects limit. Defaults to `100`.
- **GITLAB_PROJECTS_VISIBILITY**: Set default projects visibility level. Possible values `public`, `private` and `internal`. Defaults to `private`.
- **GITLAB_RESTRICTED_VISIBILITY**: Comma seperated list of visibility levels to restrict non-admin users to set. Possible visibility options are `public`, `private` and `internal`.
- **GITLAB_BACKUPS**: Setup cron job to automatic backups. Possible values `disable`, `daily` or `monthly`. Disabled by default.
- **GITLAB_BACKUP_EXPIRY**: Configure how long (in seconds) to keep backups before they are deleted. By default when automated backups are disabled backups are kept forever (0 seconds), else the backups expire in 7 days (604800 seconds).
- **GITLAB_SSH_PORT**: The ssh port number. Defaults to `22`.
- **GITLAB_RELATIVE_URL_ROOT**: The relative url of the GitLab server, e.g. `/git`. No default.
- **GITLAB_HTTPS**: Set to `true` to enable https support, disabled by default.
- **SSL_SELF_SIGNED**: Set to `true` when using self signed ssl certificates. `false` by default.
- **SSL_CERTIFICATE_PATH**: Location of the ssl certificate. Defaults to `/home/git/data/certs/gitlab.crt`.
- **SSL_KEY_PATH**: Location of the ssl private key. Defaults to `/home/git/data/certs/gitlab.key`.
- **SSL_DHPARAM_PATH**: Location of the dhparam file. Defaults to `/home/git/data/certs/dhparam.pem`.
- **CA_CERTIFICATES_PATH**: List of SSL certificates to trust. Defaults to `/home/git/data/certs/ca.crt`.
- **NGINX_MAX_UPLOAD_SIZE**: Maximum acceptable upload size. Defaults to `20m`.
- **NGINX_X_FORWARDED_PROTO**: Advanced configuration option for the `proxy_set_header X-Forwarded-Proto` setting in the GitLab nginx vHost configuration. Defaults to `https` when `GITLAB_HTTPS` is `true`, else defaults to `$scheme`.
- **REDIS_HOST**: The hostname of the Redis server. Defaults to `localhost`.
- **REDIS_PORT**: The connection port of the Redis server. Defaults to `6379`.
- **UNICORN_WORKERS**: The number of Unicorn workers to start. Defaults to `2`.
- **UNICORN_TIMEOUT**: Sets the timeout of Unicorn worker processes. Defaults to `60` seconds.
- **SIDEKIQ_CONCURRENCY**: The number of concurrent Sidekiq jobs to run. Defaults to `5`.
- **DB_TYPE**: The database type. Defaults to `postgres`.
- **DB_HOST**: The database server hostname. Defaults to `localhost`.
- **DB_PORT**: The database server port. Defaults to `3306` for mysql and `5432` for PostgreSQL.
- **DB_NAME**: The database database name. Defaults to `gitlabhq_production`.
- **DB_USER**: The database database user. Defaults to `root`.
- **DB_PASS**: The database database password. Defaults to no password.
- **DB_POOL**: The database database connection pool count. Defaults to `10`.
- **SMTP_ENABLED**: Enable mail delivery via SMTP. Defaults to `true` if `SMTP_USER` is defined, else defaults to `false`.
- **SMTP_DOMAIN**: SMTP domain. Defaults to` www.gmail.com`.
- **SMTP_HOST**: SMTP server host. Defaults to `smtp.gmail.com`.
- **SMTP_PORT**: SMTP server port. Defaults to `587`.
- **SMTP_USER**: SMTP username.
- **SMTP_PASS**: SMTP password.
- **SMTP_STARTTLS**: Enable STARTTLS. Defaults to `true`.
- **SMTP_AUTHENTICATION**: Specify the SMTP authentication method. Defaults to `login` if `SMTP_USER` is set.
- **LDAP_ENABLED**: Enable LDAP. Defaults to `false`.
- **LDAP_HOST**: LDAP Host.
- **LDAP_PORT**: LDAP Port. Defaults to `636`.
- **LDAP_UID**: LDAP UID. Defaults to `sAMAccountName`.
- **LDAP_METHOD**: LDAP method, Possible values are `ssl`, `tls` and `plain`. Defaults to `ssl`.
- **LDAP_BIND_DN**: No default.
- **LDAP_PASS**: LDAP password.
- **LDAP_ALLOW_USERNAME_OR_EMAIL_LOGIN**: If enabled, GitLab will ignore everything after the first '@' in the LDAP username submitted by the user on login. Defaults to `false` if `LDAP_UID` is `userPrincipalName`, else `true`.
- **LDAP_BASE**: Base where we can search for users. No default.
- **LDAP_USER_FILTER**: Filter LDAP users. No default.
- **OAUTH_ALLOW_SSO**: This allows users to login without having a user account first. User accounts will be created automatically when authentication was successful. Defaults to `false`.
- **OAUTH_BLOCK_AUTO_CREATED_USERS**: Locks down those users until they have been cleared by the admin. Defaults to `true`.
- **OAUTH_GOOGLE_API_KEY**: Google App Client ID. No defaults.
- **OAUTH_GOOGLE_APP_SECRET**: Google App Client Secret. No defaults.
- **OAUTH_GOOGLE_RESTRICT_DOMAIN**: Google App restricted domain. No defaults.
- **OAUTH_TWITTER_API_KEY**: Twitter App API key. No defaults.
- **OAUTH_TWITTER_APP_SECRET**: Twitter App API secret. No defaults.
- **OAUTH_GITHUB_API_KEY**: GitHub App Client ID. No defaults.
- **OAUTH_GITHUB_APP_SECRET**: GitHub App Client secret. No defaults.

## Maintenance

### Creating backups

Gitlab defines a rake task to easily take a backup of your GitLab installation. The backup consists of all git repositories, uploaded files and as you might expect, the sql database.

Before taking a backup, please make sure that the GitLab image is not running for obvious reasons:

``` bash
docker stop gitlab
```

To take a backup all you need to do is run the GitLab rake task to create a backup:

``` bash
docker run --name=gitlab -it --rm [OPTIONS] brightcommerce/gitlab:7.2.1 app:rake gitlab:backup:create
```

A backup will be created in the backups folder of the [Data Store](#data-store).

### Restoring Backups

Gitlab defines a rake task to easily restore a backup of your GitLab installation. Before performing the restore operation please make sure that the GitLab image is not running.

``` bash
docker stop gitlab
```

To restore a backup, run the image in interactive (-it) mode and pass the "app:restore" command to the container image:

``` bash
docker run --name=gitlab -it --rm [OPTIONS] brightcommerce/gitlab:7.2.1 app:rake gitlab:backup:restore
```

The restore operation will list all available backups in reverse chronological order. Select the backup you want to restore and GitLab will do its' job.

### Automated Backups

The image can be configured to automatically take backups on a daily or monthly basis. Adding `-e 'GITLAB_BACKUPS=daily'` to the docker run command will enable daily backups, while `-e 'GITLAB_BACKUPS=monthly'` will enable monthly backups.

Daily backups are created at 4am (UTC) everyday, while monthly backups are created on the 1st of every month at the same time as the daily backups.

By default, when automated backups are enabled, backups are held for a period of 7 days. While when automated backups are disabled, the backups are held for an infinite period of time. This behavior can be configured via the `GITLAB_BACKUP_EXPIRY` option.

### Shell Access

For debugging and maintenance purposes you may want access the container shell. Since the container does not allow interactive login over the SSH protocol, you can use the [nsenter](http://man7.org/linux/man-pages/man1/nsenter.1.html) linux tool (part of the util-linux package) to access the container shell.

Some linux distros (e.g. ubuntu) use older versions of the util-linux which do not include the `nsenter` tool. To get around this @jpetazzo has created a nice Docker image that allows you to install the `nsenter` utility and a helper script named `docker-enter` on these distros.

To install the nsenter tool on your host execute the following command.

``` bash
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
```

Now you can access the container shell using the command

``` bash
sudo docker-enter gitlab
```

For more information refer https://github.com/jpetazzo/nsenter

Another tool named `nsinit` can also be used for the same purpose. Please refer https://jpetazzo.github.io/2014/03/23/lxc-attach-nsinit-nsenter-docker-0-9/ for more information.

## Upgrading

GitLabHQ releases new versions on the 22nd of every month, bugfix releases immediately follow. I update this project almost immediately when a release is made (at least it has been the case so far). If you are using the image in production environments I recommend that you delay updates by a couple of days after the GitLab release, allowing some time for the dust to settle down.

To upgrade to newer GitLab releases, simply follow this 4 step upgrade procedure.

- **Step 1**: Update the Docker image:

``` bash
docker pull brightcommerce/gitlab:7.2.1
```

- **Step 2**: Stop and remove the currently running image:

``` bash
docker stop gitlab
docker rm gitlab
```

- **Step 3**: Backup the application data:

``` bash
docker run --name=gitlab -it --rm [OPTIONS] brightcommerce/gitlab:7.2.1 app:rake gitlab:backup:create
```

- **Step 4**: Start the image:

``` bash
docker run --name=gitlab -d [OPTIONS] brightcommerce/gitlab:7.2.1
```

## Rake Tasks

The `app:rake` command allows you to run GitLab rake tasks. To run a rake task simply specify the task to be executed to the `app:rake` command.

For example, if you want to gather information about GitLab and the system it runs on:

``` bash
docker run --name=gitlab -d [OPTIONS] brightcommerce/gitlab:7.2.1 app:rake gitlab:env:info
```

Similarly, to import bare repositories into GitLab project instance:

``` bash
docker run --name=gitlab -d [OPTIONS] brightcommerce/gitlab:7.2.1 app:rake gitlab:import:repos
```

For a complete list of available rake tasks please refer https://github.com/gitlabhq/gitlabhq/tree/master/doc/raketasks or the help section of your GitLab installation.

## References
  * https://github.com/gitlabhq/gitlabhq
  * https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/installation.md
  * https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/requirements.md
  * http://wiki.nginx.org/HttpSslModule
  * https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html
  * https://github.com/gitlabhq/gitlab-recipes/blob/master/web-server/nginx/gitlab-ssl
  * https://github.com/jpetazzo/nsenter
  * https://jpetazzo.github.io/2014/03/23/lxc-attach-nsinit-nsenter-docker-0-9/

## Credit

This repository was based on the work of [docker-gitlab by Sameer Naik](https://github.com/sameersbn/docker-gitlab) version 7.2.1-1. The biggest changes are the removal of Redmine and Jira connectivity options, and removal of the local installation of Redis and MySQL servers.
