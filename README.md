# How To Set Up the Latest MediaWiki with Lighttpd on Ubuntu 14.04

### Introduction

MediaWiki is a popular open source wiki platform that can be used for public or internal collaborative content publishing.  MediaWiki is used for many of the most popular wikis on the Internet, including Wikipedia, the site that the project was originally designed to serve.

In this guide, we will be setting up the latest version of MediaWiki on an Ubuntu 14.04 server.  We will use the lighttpd web server to make the actual content available, PHP-FPM to handle dynamic processing, and the MySQL database to store our wiki's data.

## Prerequisites

To complete this guide, you should have access to a clean Ubuntu 14.04 server instance.  On this system, you should have a non-root user configured with `sudo` privileges for administrative tasks.  You can learn how to set this up by following our [Ubuntu 14.04 initial server setup guide](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04).

When you are ready to continue, log into your server with your `sudo` user and get started below.

## Step One &mdash; Install the Server Components

Before we can make use of the MediaWiki software, we must install our server components first. In this section we will install the lightppd web server, the PHP scripting language with the FastCGI Process Manager (FPM), and the MySQL database.

We can install software on Ubuntu using `apt`. `Apt` is the default package manager for Debian and Debian-based Linux distributions, such as Ubuntu. A package manager is a set of software tools for managing the installation, configuration, upgrading, and removal of software packages. You can learn more about using `apt` by reading our guide on [how to manage packages in Ubuntu and Debian](https://www.digitalocean.com/community/tutorials/how-to-manage-packages-in-ubuntu-and-debian-with-apt-get-apt-cache).

`Apt` keeps track of available and installed software in a database. Before installing any software using `apt`, you will want to make sure that your local copy of the database is up-to-date so that you can install the most recent packaged versions of the software. Let's begin by updating `apt`'s local database using the `apt-get update` command. `Apt-get` requires root privileges to run. If you are logged in as an ordinary user (rather than the **root** user) on your Ubuntu droplet, you must type `sudo` before `apt-get` in order to run the command with root privileges. Type the following command and press `ENTER`:

```command
sudo apt-get update
```

If this is your first time using `sudo`, you will be prompted for a password after you enter the above command. Enter the password of the user you are logged in as. You will see a list of URLs scroll by. These are the locations from which `apt-get` retrieves information about the currently available software packages. When the list stops scrolling, your database should be up-to-date.

Now you are ready to install `lighttpd`. [Lighttpd](http://www.lighttpd.net/) (pronounced "lighty") is an open source, standards-compliant web server optimized for high-performance environments. Install `lighttpd` by entering the following command:

```command
sudo apt-get install lighttpd
```

Next, we will install `php` and `php-fpm`. The [PHP: Hypertext Processor](http://php.net/) (PHP) is an open source, server-side scripting language designed especially for web development. 

<$>[note]
**Note:** MediaWiki is written in PHP, and it will not function correctly if the `php` software is not installed.
<$>

The [FastCGI Process Manager](http://php-fpm.org/) for PHP (PHP-FPM) began as an alternative implementation of the FastCGI protocol (for using the output of a program to dynamically generate web content) for PHP, but is now included within the PHP project. It may be installed as an optional PHP module. Enter the command to install `php` and `php-fpm` on Ubuntu as follows (the "5" after "php" indicates "PHP version 5.x"):

```command
sudo apt-get install php5 php5-fpm
```

Lastly, we will install `mysql-server` and `php-mysql`. [MySQL](http://www.mysql.com/) is an open source relational database management system (RDBMS). Once you install and configure MediaWiki to use `mysql`, any edits to your wiki will be stored in your MySQL database. You will need to install the `php-mysql` module as well to enable PHP to communicate with your MySQL database. Enter the following command to install `mysql-server`:

```command
sudo apt-get install php5-mysql mysql-server
```

After you enter this command, `apt-get` will download the two packages specifed, as well as any additional packages required by `mysql-server`. While you are installing `mysql-server`, you should see the following screen prompting you to set a password for your MySQL **root** user.

![Enter a password for your MySQL root user and then select <Ok>](http://i.imgur.com/JY5ct8H.png?1)

The account for which you need to enter a password is *not* the **root** user on your Ubuntu droplet, but rather a new administrative account, also named "root", for your MySQL database. 

<$>[warning]
**Warning:** For security reasons, the password you create for your MySQL **root** user should be different from that of your Ubuntu **root** user, or any other account you use.
<$>

Go ahead and type a new password for your MySQL **root** user, and then press `ENTER`. After you enter your new password, you will be prompted to enter it again to ensure that you remember how to type it correctly. Go ahead and enter the same password again, and `mysql-server` will finish installing.

## Step Two &mdash; Configure MySQL and Create Credentials for MediaWiki

Now that MySQL is installed on your system, we need to configure it to be used securely with MediaWiki. In this section we will create a database directory structure for MySQL, secure our MySQL installation, and create a new database and database user for MediaWiki's use.

We can create a database directory structure for MySQL by running the `mysql_install_db` script. This command needs system root privileges to run. Start the script by entering the following command:

```command
sudo mysql_install_db
```

After running this command, your MySQL data directory should be initialized, and your system tables created. On Ubuntu the MySQL database directory structure should be located under `/var/lib/mysql`. The contents of this directory are viewable only by **root** and the **mysql** system user, or by non-root users with `sudo` privileges. The **mysql** system user is a special non-login account on your Ubuntu system that was created when you installed the `mysql-server` software package. It is used to perform certain tasks required by the MySQL database server.

The default MySQL installation has several features that are susceptible to exploitation by a malicious user. Fortunately, we can easily take several steps to enhance the security of our MySQL installation by running the `mysql_secure_installation` script, which also requires root privileges to run. Enter the following command to run the script:

```command
sudo /usr/bin/mysql_secure_installation
```

After you enter the command, you will be asked to enter the password you set for your MySQL **root** account. Go ahead and enter your *MySQL* root password. If you did not set a MySQL root password earlier, you should simply press `ENTER`.

Next, you will be asked if you want to change your password. If you're happy with your password, just type <^>n<^> for "no". If you did not set a password earlier, type <^>y<^> for "yes" and enter your new MySQL root password, and then enter it again at the prompt.

The following questions will ask if you want to remove anonymous users, disallow remote root logins, remove the test database and access to it, and reload the privilege tables so that these changes will take effect immediately. Type <^>y<^> for "yes" to answer each question. 

Now that our database is initialized and secured, we are ready to create a new database and user for use by MediaWiki. Let's start by logging in to MySQL as the **root** user:

```command
mysql -u root -p
```

You will be prompted to enter your MySQL root password. Enter your password. Once you are logged in, your terminal prompt will change to `mysql>`. Now that you're logged in to MySQL, let's create a database specifically for use by MediaWiki. You can choose just about any name you like for your database. In this example we will be naming our database <^>wikidb<^>. Enter the following command to create the `wikidb` database:

```custom_prefix(mysql>)
CREATE DATABASE <^>wikidb<^>;
```

If you wish to name your database something else, you may change the text in red to whatever you want it to be. You should see the following output after your database is created:

```
[secondary_label Output]
Query OK, 1 row affected (0.00 sec)
```

<$>[note]
**Note:** We'll be capitalizing MySQL commands for the sake of convention, but you may type them in lowercase if you prefer. However, for most MySQL commands, don't forget to end the line with a semicolon; otherwise MySQL will assume you haven't completed your command. If you enter a command and forget to end it with a semicolon, you may see the following prompt: `->`. Enter `;` and you will be returned to the regular MySQL prompt.
<$>

Now we'll create a MySQL user and grant it various privileges to the MediaWiki database. Although MediaWiki can use your MySQL **root** account to access the database, for security it's better to create a normal MySQL user specifically for this purpose. Again, you can choose pretty much any name, but for this example we'll be using "wikiuser" for our MySQL username. You will need to create a new password for this user. Type the following command:

```custom_prefix(mysql>)
GRANT INDEX, CREATE, SELECT, INSERT, UPDATE, DELETE, ALTER, LOCK TABLES ON <^>wikidb<^>.* TO '<^>wikiuser<^>'@'localhost' IDENTIFIED BY '<^>password<^>';
``` 

You may substitute your own preferences for the values in red. The first should be the name of the database you created in the previous command, the second should be the MySQL username you want to use, and the third should be a secure password. Be sure to remember these three values, because you will need them when you configure MediaWiki. After entering the above command, you should see the following output:

```
[secondary_label Output]
Query OK, 0 rows affected (0.00 sec)
```

Enter the following command to make the above changes take effect: 

```custom_prefix(mysql>)
FLUSH PRIVILEGES;
```

You should see the following output:

```
[secondary_label Output]
Query OK, 0 rows affected (0.00 sec)
```

Now that you've created your `wikidb` MediaWiki database, you can see it using the following command:

```custom_prefix(mysql>)
SHOW DATABASES;
```

The output should look something like this:

```
[secondary_label Output]
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| wikidb             |
+--------------------+
4 rows in set (0.00 sec)
```

You can see your new **wikiuser** MySQL user using the following command:

```custom_prefix(mysql>)
SELECT USER, HOST FROM mysql.user;
```

The output should look something like this:

```
[secondary_label Output]
+------------------+-----------+
| USER             | HOST      |
+------------------+-----------+
| root             | 127.0.0.1 |
| root             | ::1       |
| debian-sys-maint | localhost |
| root             | localhost |
| wikiuser         | localhost |
+------------------+-----------+
5 rows in set (0.00 sec)
```

When you are done using MySQL, type `quit` or `exit` to return to the Linux command line:

```custom_prefix(mysql>)
exit
```

## Step Three &mdash; Configure PHP-FPM and Lighttpd

Now that MySQL is configured for use with MediaWiki, we need to edit a few files to allow the lightppd web server and PHP-FPM to work together. First, if we need to, we can edit `/etc/php5/fpm/php.ini` to  change some defaults for PHP-FPM. Next, we need to edit `/etc/lighttpd/conf-available/15-fastcgi-php.conf` to configure PHP-FPM support in `lighttpd`. Then we need run a couple of scripts to make our changes take effect. Finally, we need to restart `lighttpd` and test PHP-FPM using our web browser. In addition, we may install some extra PHP modules to provide extra functionality for MediaWiki.

Our web server should have started when we installed `lighttpd`. Enter the IP address of your Ubuntu droplet into the address bar of your web browser. If you don't recall your droplet's IP address, a quick way to find it is by entering `hostname -I` on your droplet's command line:

```command
hostname -I
```

You should see output similar to the following (the numbers will be different):

```
[secondary_label Output]
<^>111.111.111.111 10.111.111.111<^>
```
The first IP address (<^>111.111.111.111<^> in this example) should be your public IP address. Copy it into your browser and press `ENTER`. You should see `lighttpd`'s default placeholder page, which looks something like this:

![Placeholder page &mdash; The owner of this web site has not put up any web pages yet. Please come back later.](http://i.imgur.com/pGQVsNl.png)

If you are unable to connect to your droplet, first double-check to make sure that you entered your IP address correctly. If you did, `lighttpd` is probably not running. You can start `lighttpd` using the following command:

```command
sudo service lighttpd start
```

Now that we've made sure `lighttpd` is running, let's configure PHP-FPM. PHP-FPM's main configuration file on Ubuntu 14.04 is `/etc/php5/fpm/php.ini`. PHP-FPM loads the settings in this file each time it starts. Before you make any changes to the default configuration file, it's a good idea to save a backup copy. It requires root privileges to create or modify files in the `/etc` directory, so we need to use `sudo`. Enter the following command to backup PHP-FPM's configuration file:

```command
sudo cp /etc/php5/fpm/php.ini /etc/php5/fpm/php.ini.bak
```

This way, you will have your original configuration file to refer to when needed, and if you ever break something and don't know how to fix it, you can restore your orginal configuration by entering the above command with the two filenames switched. 

We will be using `nano` to edit this configuration file, but if you have another preferred text editor, you may use that instead. Keep in mind that you may need to install it using `apt-get` if it doesn't come with the default Ubuntu 14.04 distribution. Open the PHP-FPM configuration file in your text editor:

```command
sudo nano /etc/php5/fpm/php.ini
```

The `/etc/php5/fpm/php.ini` configuration file has many lines of text, so you will need to use a search function in your text editor to find the relevant line or lines. In `nano` you can use `CTRL-W` to find text. The first option we may want to edit is the `upload_max_filesize` option. This option is useful if you want to allow other people to be able to upload images or other large files to your wiki, or if you want to be able to upload images directly to your wiki from a remote computer (that is, any computer system that is not your droplet). By default, PHP-FPM limits the `upload_max_filesize` option to 2 MB:

```
[secondary_label /etc/php5/fpm/php.ini]
[...]
; Maximum allowed size for uploaded files.
; http://php.net/upload-max-filesize
upload_max_filesize = 2M
[...]
```

If this limit is fine with you, you don't need to change it. However, many images these days are larger than 2 MB, so you may wish to change the limit to something larger. Keep in mind, however, that the larger a file is, the more disk space and bandwidth it uses. You will be the one paying for it, not the people who are uploading large files to your wiki. Let's change the limit to 8 MB, which should be plenty for most image files. If you are expecting other wiki users to upload videos, and you're willing to pay for the storage and bandwidth, you may wish to make the limit even higher. Type `CTRL-W` in `nano`, and then `filesize` to find the line with the `upload_max_filesize` option, then change the value to <^>8M<^> (or whatever you prefer):

```
[secondary_label /etc/php5/fpm/php.ini]
[...]
; Maximum allowed size for uploaded files.
; http://php.net/upload-max-filesize
upload_max_filesize = <^>8M<^>
[...]
```

You should also check to make sure file uploads are enabled. If they are enabled, you will have the following block of text in your `/etc/php5/fpm/php.ini` file:

```
[secondary_label /etc/php5/fpm/php.ini]
[...]
; Whether to allow HTTP file uploads.
; http://php.net/file-uploads
file_uploads = On
[...]
```

This block of text (or "stanza") should be two stanzas above the `upload_max_filesize` stanza. If you *don't* want to allow other people to upload files to your wiki, and you don't need to be able to upload images to your wiki from a remote computer, you can disable file uploads by inserting a semicolon before the line that reads `file_uploads = On`:

```
[secondary_label /etc/php5/fpm/php.ini]
[...]
; Whether to allow HTTP file uploads.
; http://php.net/file-uploads
<^>;<^>file_uploads = On
[...]
```

When PHP-FPM reads the `/etc/php5/fpm/php.ini` configuration file, it ignores any line that begins with a semicolon. Save the file by typing `CTRL-O` and `ENTER`, and exit `nano` by typing `CTRL-X`

Now we're ready to configure `lighttpd` to use PHP-FPM. We will need to edit the file located at `/etc/lighttpd/conf-available/15-fastcgi-php.conf`. This file contains PHP-FastCGI settings for use with `lighttpd`. First, make a backup of the file by typing the following command:

```command
sudo cp /etc/lighttpd/conf-available/15-fastcgi-php.conf /etc/lighttpd/conf-available/15-fastcgi-php.conf.bak
```

Now open the original file in your text editor using the following command:

```command
sudo nano /etc/lighttpd/conf-available/15-fastcgi-php.conf
```

Your file should look something like this:

```
[secondary_label /etc/lighttpd/conf-available/15-fastcgi-php.conf]
# -*- depends: fastcgi -*-
# /usr/share/doc/lighttpd-doc/fastcgi.txt.gz
# http://redmine.lighttpd.net/projects/lighttpd/wiki/Docs:ConfigurationOptions#mod_fastcgi-fastcgi

## Start an FastCGI server for php (needs the php5-cgi package)
fastcgi.server += ( ".php" => 
	((
		"bin-path" => "/usr/bin/php-cgi",
		"socket" => "/var/run/lighttpd/php.socket",
		"max-procs" => 1,
		"bin-environment" => ( 
			"PHP_FCGI_CHILDREN" => "4",
			"PHP_FCGI_MAX_REQUESTS" => "10000"
		),
		"bin-copy-environment" => (
			"PATH", "SHELL", "USER"
		),
		"broken-scriptfilename" => "enable"
	))
)
```

This configuration file is for use with PHP's older FastCGI implementation. PHP-FPM really only needs to use two variables from this file: `socket` and `broken-scriptfilename`. You may comment out or remove all the other lines between the double parentheses. You will also need to change the value of `"socket"` from `"/var/run/lighttpd/php.socket"` to `"/var/run/php5-fpm.sock"`.

In `/etc/lighttpd/conf-available/15-fastcgi-php.conf` you can comment out a line by putting a hash (`#`, also known in North America as a "pound sign") at the beginning. When `lighttpd` runs and reads this configuration file, it ignores any line that begins with `#`. In `nano` you can remove a line by typing `CTRL-K`. Go ahead and edit the file as described above, and then save the file and exit `nano`. When you finish, the section between the double parentheses should appear like this:

```
[secondary_label /etc/lighttpd/conf-available/15-fastcgi-php.conf]
[...]
	((
<^>#<^>		"bin-path" => "/usr/bin/php-cgi",
		"socket" => "<^>/var/run/php5-fpm.sock<^>",
<^>#<^>		"max-procs" => 1,
<^>#<^>		"bin-environment" => ( 
<^>#<^>			"PHP_FCGI_CHILDREN" => "4",
<^>#<^>			"PHP_FCGI_MAX_REQUESTS" => "10000"
<^>#<^>		),
<^>#<^>		"bin-copy-environment" => (
<^>#<^>			"PATH", "SHELL", "USER"
<^>#<^>		),
		"broken-scriptfilename" => "enable"
	))
[...]
```

or like this:

```
[secondary_label /etc/lighttpd/conf-available/15-fastcgi-php.conf]
[...]
	((
		"socket" => "<^>/var/run/php5-fpm.sock<^>",
		"broken-scriptfilename" => "enable"
	))
[...]
```

<$>[note]
**Note:** Be certain at least to disable or remove the `bin-path` line. The file `/usr/bin/php-cgi` may not exist on your system, PHP-FPM doesn't need it to run, and if the line is not disabled, you may have trouble connecting to your web server.
<$>

Now that we've edited `lighttpd`'s PHP-FastCGI configuration file to use PHP-FPM, we need to run a couple of scripts to make sure `lighttpd` loads its FastCGI and PHP-FastCGI configuration files when it runs. The file we just edited is located in the `/etc/lighttpd/conf-available` directory. Files in this directory will be ignored when `lightppd` runs, but files in the `/etc/lighttpd/conf-enabled` directory will be loaded. Enable FastCGI in `lightppd` by entering the following commands:

```command
sudo lighttpd-enable-mod fastcgi
```

and

```command
sudo lighttpd-enable-mod fastcgi-php
```

These commands create *symbolic links* (or shortcuts) in the `/etc/lighttpd/conf-enabled` directory to the `10-fastcgi.conf` and `15-fastcgi-php.conf` files in the `/etc/lighttpd/conf-available` directory. You can check for the files by entering the following command:

```command
ls -l /etc/lighttpd/conf-enabled/
```

The output should look something like this:

```
[secondary_label Output]
total 0
lrwxrwxrwx 1 root root 33 May 13 17:28 10-fastcgi.conf -> ../conf-available/10-fastcgi.conf
lrwxrwxrwx 1 root root 37 May 13 17:28 15-fastcgi-php.conf -> ../conf-available/15-fastcgi-php.conf
```

This shows that `10-fastcgi.conf` and `15-fastcgi-php.conf` are links in the `/etc/lighttpd/conf-enabled/` directory, pointing to actual files of the same name in the `/etc/lighttpd/conf-available` directory.

Now that we've fully configured `php5-fpm` and `lighttpd`, lets restart both servers:

```command
sudo service php5-fpm restart
```

```command
sudo service lighttpd restart
```

We can check if PHP-FPM is working by creating a PHP file in `/var/www`. On Ubuntu the Lighttpd web server uses the `/var/www/` directory as its default *document root*. The document root is a directory used for storing web pages and other files that are made accessible to the public through the web server. There should already be an HTML file, `index.lighttpd.html`, in your document root. This is the placeholder page created when you installed `lighttpd`. Now, let's use our text editor to create a PHP script called `info.php` to test out PHP-FPM:

```command
sudo nano /var/www/info.php
```

Add the following text and save the file:

```
[secondary_label /var/www/info.php]
<^><?php<^>
<^>phpinfo();<^>
<^>?><^>
```

We can test this script by pointing our web browser to `http://<^>111.111.111.111<^>/info.php`. (Replace "111.111.111.111" with your droplet's IP address.) You should see something like this:

![PHP Version 5.5.9-1ubuntu4.9 ... six .ini files parsed](http://i.imgur.com/vAGlJ3J.png)

You're almost ready to install MediaWiki. However, before we go on, let's install a few additional PHP modules to extend the functionality available to MediaWiki. You can see the PHP modules available in Ubuntu using `apt-cache`:

```command
apt-cache search php5-
```

You should see a list of PHP modules with short descriptions after you enter this command. You can find out more about a particular module using `apt-cache show php5-<^>module-name<^>`:

```command
apt-cache show php5-<^>gd<^>
```

The output should look something like this (with the less relevant information snipped):

```
[secondary_label Output]
[...]
Installed-Size: 159
[...]
Depends: libc6 (>= 2.4), libgd3 (>= 2.1.0~alpha~), libxpm4, phpapi-20121212, php5-common (= 5.5.9+dfsg-1ubuntu4.9), ucf
Pre-Depends: dpkg (>= 1.15.7.2~)
Filename: pool/main/p/php5/php5-gd_5.5.9+dfsg-1ubuntu4.9_amd64.deb
Size: 28014
[...]
Description-en: GD module for php5
 This package provides a module for handling graphics directly from PHP
 scripts. It supports the PNG, JPEG, XPM formats as well as Freetype/ttf fonts.
[...]
```

The first section, `Installed-Size`, shows how much space in kilobytes the software package will use on your system after it has been installed. The second section shows what other software must be installed before this package can be installed (`Depends`), and how much space in kilobytes it will all use on your system after it all has been installed (`Size`). The last section, `Description-en`, gives a description of what this particular software package is used for. You may install any modules you think you will need, but in this tutorial we will be installing the following PHP5 modules for use by MediaWiki:

* **php5-intl:** Provides internationalization support. Install it if you want wiki users to be able to create articles in languages other than the most common European languages.
* **php5-gd:** Provides support for image thumbnailing. You may install `php5-imagick`, which provides similar functionality, if you prefer.
* **php5-xcache:** Optimizes the performance of PHP scripts.

Enter the following command to install the modules:

```commmand
sudo apt-get install php5-intl php5-gd php5-xcache
```

Reload PHP-FPM so the new modules will be available:

```command
sudo service php5-fpm restart
```

Now point your web browser to `http://<^>111.111.111.111<^>/info.php` (remember to use your droplet's IP address), or reload the page if you are already there. This time the page should be longer, and the "Additional .ini files parsed" table should have additional entries:

![PHP Version 5.5.9-1ubuntu4.9 ... nine .ini files parsed](http://i.imgur.com/DWGbBW8.png)

As you can see, the `info.php` file contains quite a bit of information about your system. It is also publicly accessible and easy to find for anyone with a bit of Internet savvy. Now that you have tested PHP-FPM, it's a good idea to remove the `info.php` file. Type this command to remove `info.php`:

```command
sudo rm /var/www/info.php
```

## Step Four &mdash; Install MediaWiki

Now that we have installed and configured all our server components, and tested them to make sure they are functioning, we are ready to install MediaWiki. We could use `apt-get` to install MediaWiki, but unfortunately the version of MediaWiki available through Ubuntu's software repository is rather outdated, so instead we will be downloading the latest stable version of MediaWiki directly from the MediaWiki website.

At the time of writing this tutorial the latest stable version of MediaWiki is 1.24.2. You can find the latest stable version of MediaWiki by pointing your web browser to the MediaWiki [Download page](http://www.mediawiki.org/wiki/Download). There you will find a link to the latest stable version of the software. The link should point to a URL that looks something like this: [http://releases.wikimedia.org/mediawiki/<^>1.24<^>/mediawiki-<^>1.24.2<^>.tar.gz](http://releases.wikimedia.org/mediawiki/1.24/mediawiki-1.24.2.tar.gz). We want to download this software to our `sudo` user's home directory on our droplet.

Your home directory should look something like `/home/<^>sammy<^>`, where **sammy** is your username. You can show what directory you are in using the `pwd` ("print working directory") command:

```command
pwd
```

If you are in your home directory, you should see the following output:

```
[secondary_label Output]
/home/<^>sammy<^>
```

If you are not in your droplet user's home directory, you can change to your home directory by entering the `cd` (change directory) command:

```command
cd
```

By default, `cd` changes your current directory to your home directory.

```[note]
**Note:** If you are in your home directory, and you are using Ubuntu's default `bash` shell prompt, your prompt should look something like `<^>sammy<^>@<^>florin<^>:~$`, where **sammy** is your username and **florin** is your droplet's hostname. The tilde (~) indicates your home directory.
```

Now that we have made certain that we're in our home directory, let's download the MediaWiki software package. In order to download this software package to your droplet, you will need to use a command-line downloader such as `wget` or `curl`. On your droplet's command line, type "wget" followed by the URL of the latest stable MediaWiki version:

```command
wget http://releases.wikimedia.org/mediawiki/<^>1.24<^>/mediawiki-<^>1.24.2<^>.tar.gz
```

Make sure to modify the parts of the URL highlighted in red according to the most recent stable release of MediaWiki.

Now that we have the MediaWiki software package in our home directory, let's extract it using the `tar` command:

```command
tar xvzf mediawiki-<^>1.24.2<^>.tar.gz
```

This will create a directory named `mediawiki-<^>1.24.2<^>` under your home directory. We want to place the contents of this directory in a place that is publicly accessible through our web server. You will recall that the lighttpd web server's document root is `/var/www`. If we move the contents of the MediaWiki directory to `/var/www`, you will be able to access your wiki simply by entering `http://<^>111.111.111.111<^>` (replace the text in red with the IP address of your droplet) into your web browser's address bar. Article pages will appear in the form `http://<^>111.111.111.111<^>/index.php?title=<^>Article<^>`. However, the MediaWiki project recommends against putting the MediaWiki files directly in the document root because they will conflict with other files and directories located in the document root. Instead, you may place the files in a new subdirectory of the document root, such as `/var/www/mediawiki` or `/var/www/w`. Article pages would then appear as `http://<^>111.111.111.111<^>/mediawiki/index.php?title=<^>Article<^>` or `http://<^>111.111.111.111<^>/w/index.php?title=<^>Article<^>`, respectively. You can also move the files to another location on your system, such as `/home/<^>sammy<^>/public_html/<^>mediawiki<^>`, `/usr/share/<^>mediawiki<^>`, or `/var/lib/<^>mediawiki<^>`, and then create a symbolic link to the directory from `/var/www/<^>mediawiki<^>`. For the purpose of this tutorial, we will be moving the files to `/usr/share/mediawiki` and creating a symbolic link from `/var/www/mediawiki`, but you may simply move the files to `/var/www/<^>mediawiki<^>`, if you prefer. Let's start by moving the files to `/usr/share/mediawiki`:

```command
mv mediawiki-<^>1.24.2<^> <^>/usr/share/mediawiki<^>
```

Now create a symbolic link from `/var/www/mediawiki` to `/usr/share/mediawiki`:

```command
ln -s <^>/usr/share/mediawiki/<^> /var/www/<^>mediawiki<^>
```

```[note]
**Note:** The first argument to the `ln -s` command is the path to the existing file or directory we want to link to, and the second argument is the path to the link we are creating. Make sure you use full path names with this command.
```

Now MediaWiki should be Internet-accessible. However, we still need to run MediaWiki's configuration script to set up our wiki. Point your web browser to `http://<^>111.111.111.111<^>/<^>mediawiki<^>/mw-config/index.php` (where <^>mediawiki<^> is the directory or symbolic link under `/var/www` that you created previously) to begin creating the configuration file. You should be taken to a page that is titled **MediaWiki <^>1.24.2<^> installation** and allows you to select languages for the installation process, and for the wiki. This is the first page of the installer. Select your languages and click **Continue**. 

The next page is titled **Welcome to MediaWiki!** If you have installed the software needed by MediaWiki, you should see a line of green text that reads: **The environment has been checked. You can install MediaWiki.** Click **Continue**

In the **Database settings** page, you can leave the settings as is and continue on. In the **Connect to database** page you need to enter the settings you used when you set up MySQL. We created a database named `wikidb` and a MySQL user named **wikiuser**, and set a password for **wikiuser**. Enter these three values as your **Database name**, **Database username**, and **Database password**, as in the image below:

![MySQL settings &mdash; Database name: wikidb, Database username: wikiuser, Database password: ************](http://i.imgur.com/kjXPq04.png)

Continue on to the **Name** page. Here you can name your wiki and set up the administrator account. Choose a suitable name for your wiki, give your administrator account a name (e.g. "Admin"), a strong password, and a valid contact e-mail address. Make sure to remember your account name and password because you will need them to log in to your wiki. At this point you are given the option of completing the setup or continuing on to the next page. Contiue on to the next page.

The **Options** page has several settings you may want to change. First, the default emergency contact e-mail address is something like `apache@<^>111.111.111.111<^>`. Change this to a valid e-mail address from which you can send mail. Second, check **Enable file uploads** if you want to be able to upload images or other files to your wiki from your own computer. Also, if you have a logo for your wiki, you can enter its location under **Logo URL**. Lastly, under **Settings for object caching**, select "PHP object caching (APC, XCache or WinCache)", since we installed XCache earlier. 

When you are finished setting up your options, click **Continue**. On the **Install** page, you will be given the option of going back to make more changes or continuing with the installation. If you are satisfied with your choices, click **Continue**. You will see a list of tasks performed by the installation script, and then you will be taken to the **Complete!** page, where the MediaWiki configuration file, `LocalSettings.php`, will begin downloading to your local computer. Make sure the file has been fully downloaded to your computer before you close out of the **Complete!** page.

In order to finish setting up MediaWiki, we need to copy this file to the MediaWiki directory on our droplet. We can do this by selecting and copying the file's contents and pasting them into a text editor on our droplet. Begin by opening the LocalSettings.php file in a text editor on your local computer. Select the entire contents of the file and copy them. Now open a new file named `<^>/usr/share/mediawiki/<^>LocalSettings.php` in a text editor on your droplet:

```command
sudo nano <^>/usr/share/mediawiki/<^>LocalSettings.php
```

Finally, paste the contents of the file into your editor and save the new file. Now you can enter your new wiki. Point your browser to `http://<^>111.111.111.111<^>/<^>mediawiki<^>` (substituting your own droplet's IP address and MediaWiki directory) and you should be taken to the main page of your wiki. Click on the login link in the upper right-hand corner of the page and enter the administrator account name and password you chose during the MediaWiki installation. Once you are logged in, you can create a new article by typing its name into the search bar on your wiki. If the page does not already exist, you will be asked if you want to create a new page on your wiki. Go ahead and start editing.

## Step Five &mdash; Configure MediaWiki for Short URLS

If you followed the instructions above, the article URLs in your wiki should appear as `http://<^>111.111.111.111<^>/mediawiki/index.php?title=<^>Article<^>`. That's a bit long. In this section we will configure our wiki to use URLs of the form `http://<^>111.111.111.111<^>/wiki/<^>Article<^>`, the same configuration Wikipedia uses. First, open the `LocalSettings.php` file in your text editor:

```command
sudo nano <^>/usr/share/mediawiki/<^>LocalSettings.php
```

Go to the end of the file and enter the following two lines:

```
[secondary_label LocalSettings.php]
<^>$wgArticlePath = "/wiki/$1";<^>
<^>$wgUsePathInfo = false;<^>
```

The first line tells MediaWiki to use URLs of the form `.../wiki/<^>Article<^>`. The second line tells MediaWiki not to use the `PATH_INFO` variable. This variable is only needed for URLs of the form `.../index.php/Article` and, since that doesn't apply to our setup, it's best to disable it. Save the LocalSettings.php file and exit your text editor.

We need to configure `lighttpd` as well. Open `/etc/lighttpd/lighttpd.conf` in your text editor:

```command
sudo nano /etc/lighttpd/lighttpd.conf
```

At the beginning of the file, there is a stanza that looks like this:

```
[secondary_label /etc/lighttpd/lighttpd.conf]
server.modules = (
        "mod_access",
        "mod_alias",
        "mod_compress",
        "mod_redirect",
#       "mod_rewrite",
)
[...]
```

Remove the hash from in front of `mod_rewrite" so it looks like this:

```
[secondary_label /etc/lighttpd/lighttpd.conf]
server.modules = (
        "mod_access",
        "mod_alias",
        "mod_compress",
        "mod_redirect",
        "mod_rewrite",
)
[...]
```

Now go to the end of the file and enter the following lines of text:

```
[secondary_label /etc/lighttpd/lighttpd.conf]
[...]
url.rewrite-once = (
        "^/wiki(/|$)" => "/<^>mediawiki<^>/index.php",
        "^/$" => "/<^>mediawiki<^>/index.php",
)
```

Replace <^>mediawiki<^> with the directory or symbolic link under `/var/www` where the MediaWiki files are located, if you used a different name. The second line above tells `lighttpd` to point URLs in the form `http://<^>111.111.111.111<^>/wiki/<^>Article<^>` to articles in `/var/www/<^>mediawiki<^>`. The third line tells `lighttpd` send web browsers to your wiki's home page when they try to access your web server's document root.

Now point your Web browser to `http://<^>111.111.111.111<^>` (replace with your droplet's IP address). You should be sent to `http://<^>111.111.111.111<^>/wiki/Main_Page`.

## Conclusion

In this tutorial we have learned how to install and configure the lighttpd web server, PHP-FPM, and the MySQL database. We have also learned how to download and setup the latest version of MediaWiki. In addition, we have configured MediaWiki to use short URLs. You can learn more about configuring MediaWiki by reading our [tutorial](https://www.digitalocean.com/community/tutorials/how-to-customize-mediawiki-using-the-localsettings-php-file) on the LocalSettings.php file. You can also learn more about using and customizing MediaWiki from the links in the **Getting Started** section of your wiki's Main Page.
