# Setup Nextcloud with Portainer and nginx proxy manager

Hello, today I'm going to teach how to install Nextcloud on a Debian machine, using Portainer to manage all your docker containers and nginx proxy manager, to have SSL connection.

For those who don't know about Nextcloud, I highly recommend taking a look at their [page](https://nextcloud.com/). It is an open source software that enables you to handle your data, for those who have concerns about privacy, and do not trust google, apple ... Nextcloud provides a way to you store your files and photos, manage your calendar, save contacts and much more... And it is free.

I will provide a GitHub with all the files you need to set up, your Nextcloud instance.

For this tutorial, I'm assuming that you have already a domain and ave set up your subdomains in your VPS manager.

Before we start, you need to open connections for ports, in your firewall.
- 81
- 9000
- 8080

After we set up nginx, we can close the connection on those ports and use only the subdomain that we will set up on nginx proxy manager.

## Setup env files.

### Nextcloud

1. Open the folder ***//nextcloud**/, and edit the file **/.env**/:

Define values for all the following attrs:

```
{user_password}
{db_user}
{db_password}

{redis_password}

{nc_admin_user}
{nc_admin_password}
{your.domain.com}

{timezone} # America/Winnipeg
```

2. Edit the file **/docker-compose.yml**/ inside the folder **//nextcloud**/;

```
{redis_password}
```

### Nginx proxy manager

1. Open the folder **//npm**/, and edit the file **/.env**/;

Define values for all the following attrs:

```
{npm_db_password}
{npm_db_name}
{npm_db_user}
{npm_db_password}
```

Here we also deploy a MariaDB container, but this is only for nginx proxy manager to persist its data.

## Installing Portainer

First we need to install **/docker**/, **/docker-compose**/ and **/portainer**/ on our machine.

Connect via ssh or use **/FileZilla**/ or **/putty**/ to upload the file. In this step, we will install all dependencies on your machine to run docker and Portainer.

1. Upload the script **/setup.sh**/ to your machine;

2. Run the following commands:

- To give permission to the file to be executable.
> chmod +x setup.sh

- Run the script to install all the dependencies, be aware that you will need to accept some terms.
> ./setup.sh

3. Wait until the installation finishes.

4. Open in your browser **/{your_vps_ip}:9000**/ and set up an admin account for Portainer.

## Setup portainer agent

Portainer agent add some more functionality to your portainer CE.

1. Go to Environments in you portainer and add a new one;

2. in **Environment URL** you will add your **{host_ip}:9001**;

3. In case that you have setup and HTTPS domain you can add it to public ip;

4. Add a name and create the environment.

Some of the features that portainer agent enables is, to browse you volumes through the interface, without the need to connect via FTP or SSH to change files for example.

## Installing nginx proxy manager

Nginx proxy manager it is a container with an application that makes it easier to configure nginx, if you have knowledge and experience with nginx you can also configure using him directly, this is just a friendly UI.

1. In you Portainer;

2. Go to Stacks → + add stack → upload. 
  1. Select the file from my repository **/npm/docker-compose.yml**/;
  2. Also upload the .env file.

3. Deploy the stack that you have created.

4. Open **/{your_vps_ip}:81**/, log with the following account and change your password after it.

```text
admin@example.com
changeme
```

Here we will configure three domains, example:
- portainer.rodrigo.com.br → To access your Portainer application;
- drive.rodrigo.com.br → To access your Nextcloud application;
- proxy.rodrigo.com.br → To access your nginx proxy manager application;
- office.rodrigo.com.br → To access your onlyoffice instance (you will not access directly this URL, nextcloud will use it);

Check the image on **/images/npm_hosts_example**/ to see how it needs to be after you create the four hosts.

5. Go to hosts → Proxy hosts and create a new one, this configuration is valid for all the four hosts
    - Tab detail: Check Cache assets, block common exploits and web sockets support;
    - Tab SSL: Force SSL, HTTP/2 and HSTS enabled;

For Nextcloud, add the following lines to advanced tab, custom nginx configuration:

```
proxy_set_header Host $host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_max_temp_file_size 16384m;
client_max_body_size 0;

location = /.well-known/carddav {
  return 301 $scheme://$host:$server_port/remote.php/dav;
}

location = /.well-known/caldav {
  return 301 $scheme://$host:$server_port/remote.php/dav;
}
```

6. Now you should be able to access your Portainer and NPM application using the domains you just set up.

## Installing Nextcloud

2. Go to Stacks → + add stack → upload. 
  1. Select the file from my repository **/nextcloud/docker-compose.yml**/;
  2. Also upload the .env file.

2. Deploy the stack that you have created;

4. In your browser, open **/{your_vps_ip}:8080**/, to create the config.php file;

3. In Portainer select the Nextcloud container and click on `exec a new console`, to connect to the Nextcloud container;

Run the following command inside the **/config** folder to install vim and create a config.php file, where we will add some configurations to Nextcloud;
> apt-get update && apt-get install vim && vim config.php

4. Add the following code to the config.php file;

```php
'redis' =>
   array (
     'host' => 'redis',
     'port' => 6379,
     'password' => '{redis_password}',
  ),
  'filelocking.enabled' => true,
  'memcache.locking' => '\OC\Memcache\Redis',
  'default_phone_region' => 'BR',
  'trashbin_retention_obligation' => '30, 60',
  'overwriteprotocol' => 'https',
  'default_language' => 'pt_BR',
  'default_locale' => 'pt_BR',
  'log_type' => 'file',
  'logfile' => 'nextcloud.log',
  'logdateformat' => 'F d, Y H:i:s',
  'maintenance' => false,
```

Save and restart the Nextcloud instance;

5. Type in your browser the domain that you have set up for Nextcloud, example: nextcloud.rodrigo.com.br;

5. Set up an admin account and select **MySQL/MariaDB**
  1. Important, you need to click on **Storage & Database** to **set up MySQL/MariaDB**;
  2. Fill the database credentials, those same that you have set up before on the nextcloud.env file;
  3. The last input is the IP to connect to your **MariaDB** container;
    1. You need to go back to Portainer and get the IP that was attached to the Nextcloud MariaDB container. **{container_ip}:3306**, do not forget to add the port 3306 at the end.

6. Now your Nextcloud is up and running, congratulations.

## Setup systemd, to run scheduled jobs

Note: You may want to change the default cron time, the default is 3 minutes, you can change it inside the file **nextcloudcron.timer**, changing the variable **OnUnitActiveSec**.

1. Copy the two files inside the **/systemd** folder to **/etc/systemd/system/** of your host machine.

2. Run the following command to start the job on the system boot;
> systemctl enable nextcloud-cron.timer

2. Run the following command to start the job at once;
> systemctl start nextcloud-cron.timer

3. And run the following command to check the status;
> systemctl status nextcloud-cron.timer

If you want to run the job to generate previews, you need to run the same commands for **nextcloud-preview-generator.timer** and **nextcloud-preview-generator.service**.

## Adding only office

With onlyoffice you can have the same functionalities that you have on Google Drive to create slides, documents, sheets ...

1. Add a new stack on portainer with the name **onlyoffice** and upload the **docker-compose.yml** inside the folder **/onlyoffice**;

2. Open nextcloud with your admin account, add the onlyoffice plugin;

3. Open setting menu and click on **onlyoffice** option;

4. Add your domain to **ONLYOFFICE Docs address** (domain that you previous setup on npm, **office.rodrigo.com.br**);

5. And to finish, you can change the configuration following the image **only_office_example** on /images;

## Important notes

In case that you have problems to set up your domain at NPM, be aware that for own domain, example: nextcloud.rodrigo.com.br you have a limit of 5 certificates within 168 hours. So let's encrypt will generate only five SSL certificates, if you reach the limit you need to test with a different subdomain.

## References

1. [Deploy & Configure Nextcloud on Docker](https://techindie.net/deploy-configure-nextcloud-on-docker/)

2. [Docker - Nextcloud](https://github.com/talesam/nextcloud)

3. [Nextcloud with SSL and Docker](https://github.com/LibreCodeCoop/nextcloud-docker)

4. [Easiest Way to Deploy Your Own Nextcloud with your Own Domain Using Portainer](https://www.51sec.org/2020/11/28/easiest-way-to-deploy-your-own-nextcloud-with-your-own-domain-using-portainer/)