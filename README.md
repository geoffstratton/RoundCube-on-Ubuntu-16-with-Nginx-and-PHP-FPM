# Install Roundcube on Ubuntu 16 with Nginx and PHP-FPM

So you want to set up a webmail system. SquirrelMail was a fine community-supported project for many years, but as of July 2017 it seems to have been abandoned by its developers, so we're going to use Roundcube for our webmail system instead.

This guide assumes you have Ubuntu 16 running a [MariaDB-Postfix-Dovecot email server](https://github.com/geoffstratton/Ubuntu-Email-Server) with SpamAssassin. The instructions here were written using Roundcube version 1.3.

## Download Roundcube

Download a [complete version of Roundcube](https://roundcube.net/), and untar it into an appropriate directory, e.g., /var/www/webmail.

## Create a database for Roundcube

```sql	
MariaDB [(none)]> CREATE DATABASE webmail DEFAULT CHARACTER SET = 'utf8'; GRANT SELECT, UPDATE, INSERT, DELETE, CREATE, DROP, INDEX, ALTER, CREATE TEMPORARY TABLES ON webmail.* TO 'webmail_user'@'localhost' IDENTIFIED BY 'webmail_password';
```

## Set up Nginx to host your Roundcube site

Nginx ignores Roundcube's Apache .htaccess files, so we're telling Nginx to ignore some sensitive directories:
	
```console
root@ubuntu# vi /etc/nginx/sites-available/webmail.myserver.com
```

```nginx
server {
        listen 80;
        listen [::]:80;

        server_name webmail.myserver.com;
        root /var/www/webmail;

        index index.php;

        location ~ ^/(README.md|INSTALL|LICENSE|CHANGELOG|UPGRADING)$ {
            deny all;
        }

        location ~ ^/(config|temp|logs)/ {
            deny all;
        }

        location ~ /\. {
            deny all;
        }

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
            # With php7.0-fpm:
                fastcgi_pass unix:/run/php/php7.0-fpm.sock;
        }
}
```	

```console
root@ubuntu# ln -s /etc/nginx/sites-available/webmail /etc/nginx/sites-enabled/webmail.myserver.com
root@ubuntu# systemtcl restart nginx
```

## Configure Roundcube

In your /var/www/webmail/config directory, copy config.inc.php.sample to config.inc.php. Then edit config.inc.php, adding your database credentials from step 2 above and your IMAP and SMTP settings:

```php
// Enable the installer
$config['enable_installer'] = true;

// Set up your database connection, using the credentials you specified in step 2 above
$config['db_dsnw'] = 'mysql://username:password@localhost/database';

// Your settings from the first mailserver article.

// ----------------------------------
// IMAP
// ----------------------------------
// The mail host chosen to perform the log-in.
$config['default_host'] = 'ssl://mail.myserver.com:993';

// TCP port used for IMAP connections
$config['default_port'] = 993;

// Certificate verification options for our SSL cert
// If you're using Let's Encrypt, use these parameters instead of 'cafile':
// 'ssl_cert' => '/etc/letsencrypt/live/mail.domain1.com/fullchain.pem',
// 'ssl_key' => '/etc/letsencrypt/live/mail.domain1.com/privkey.pem',
$config['imap_conn_options'] = array(
'ssl'         => array(
    'verify_peer'  => true,
    'verify_peer_name' => false,
    'verify_depth' => 3,
    'cafile'       => '/etc/ssl/certs/dovecot.pem',
),
);

// SMTP server
$config['smtp_server'] = 'tls://mail.myserver.com';

// SMTP port (default is 25; use 587 for STARTTLS or 465 for the
// deprecated SSL over SMTP (aka SMTPS))
$config['smtp_port'] = 587;

// SMTP username (if required) if you use %u as the username Roundcube
// will use the current username for login
$config['smtp_user'] = '%u';

// SMTP password (if required) if you use %p as the password Roundcube
// will use the current user's password for login
$config['smtp_pass'] = '%p';

$config['smtp_auth_type'] = 'PLAIN';

$config['smtp_auth_cid'] = null;

$config['smtp_auth_pw'] = null;

$config['smtp_helo_host'] = '';

$config['smtp_timeout'] = 0;

// If you're using Let's Encrypt, use these parameters instead of 'cafile':
// 'ssl_cert' => '/etc/letsencrypt/live/mail.domain1.com/fullchain.pem',
// 'ssl_key' => '/etc/letsencrypt/live/mail.domain1.com/privkey.pem',
$config['smtp_conn_options'] = array (
'ssl' =>
array (
    'verify_peer' => true,
    'verify_peer_name' => false,
    'verify_depth' => 3,
    'cafile' => '/etc/ssl/certs/dovecot.pem',
),
);
```

## Run the installer

In your web browser, visit http://<span></span>webmail.myserver.<span></span>com/installer as specified by your Nginx setup. If you configured everything correctly, Roundcube will walk you through its checks and configuration settings.

On screen 1, click the "initialize database" button to set up Roundcube's tables.

```shell	
Checking PHP version

Version:  OK(PHP 7.0.18-0ubuntu0.16.04.1 detected)

Checking PHP extensions

The following modules/extensions are required to run Roundcube:

PCRE:  OK
DOM:  OK
Session:  OK
XML:  OK
JSON:  OK
PDO:  OK
Multibyte:  OK
OpenSSL:  OK

The next couple of extensions are optional and recommended to get the best performance:

FileInfo:  OK
Libiconv:  OK
Intl:  NOT AVAILABLE(See http://www.php.net/manual/en/book.intl.php)
Exif:  OK
LDAP:  NOT AVAILABLE(See http://www.php.net/manual/en/book.ldap.php)

Checking available databases

Check which of the supported extensions are installed. At least one of them is required.

MySQL:  OK
PostgreSQL:  NOT AVAILABLE(See http://www.php.net/manual/en/ref.pdo-pgsql.php)
SQLite:  NOT AVAILABLE(See http://www.php.net/manual/en/ref.pdo-sqlite.php)
SQLite (v2):  NOT AVAILABLE(See http://www.php.net/manual/en/ref.pdo-sqlite.php)
SQL Server (SQLSRV):  NOT AVAILABLE(See http://www.php.net/manual/en/ref.pdo-sqlsrv.php)
SQL Server (DBLIB):  NOT AVAILABLE(See http://www.php.net/manual/en/ref.pdo-dblib.php)
Oracle:  NOT AVAILABLE(See http://www.php.net/manual/en/book.oci8.php)

Check for required 3rd party libs

This also checks if the include path is set correctly.

PEAR:  OK
Auth_SASL:  OK
Net_SMTP:  OK
Net_IDNA2:  OK
Mail_mime:  OK
Net_LDAP3:  OK

Checking php.ini/.htaccess settings

The following settings are required to run Roundcube:

file_uploads:  OK
session.auto_start:  OK
mbstring.func_overload:  OK
suhosin.session.encrypt:  OK

The following settings are optional and recommended:

allow_url_fopen:  OK
date.timezone:  OK

Check config file

defaults.inc.php:  OK
config.inc.php:  OK

Check if directories are writable

Roundcube may need to write/save files into these directories

/var/www/webmail/temp/:  OK
/var/www/webmail/logs/:  OK

Check DB config

DSN (write):  OK
DB Schema:  NOT OK(Database not initialized)

[[Initialize database]]

Test filetype detection

Fileinfo/mime_content_type configuration:  OK
Mimetype to file extension mapping:  OK
```

On screen 2, you shouldn't need to change anything:

```shell	
General configuration
product_name

Roundcube Webmail
The name of your service (used to compose page titles)
support_url

Provide an URL where a user can get support for this Roundcube installation.
PLEASE DO NOT LINK TO THE ROUNDCUBE.NET WEBSITE HERE!
Enter an absolute URL (including http://) to a support page/form or a mailto: link.

skin_logo

Custom image to display instead of the Roundcube logo.
Enter a URL relative to the document root of this Roundcube installation.

temp_dir

/var/www/webmail/temp/
Use this folder to store temp files (must be writeable for webserver)
des_key

qCsBienDEAdkfeiasefea14BR
This key is used to encrypt the users imap password before storing in the session record
It's a random generated string to ensure that every installation has its own key.

ip_check
Check client IP in session authorization
This increases security but can cause sudden logouts when someone uses a proxy with changing IPs.

enable_spellcheck
Make use of the spell checker

etc.
```

On screen 3, you can test your connection by using a valid email user on your system.

## Clean up

Disable the installer in config.inc.php and remove the installer folder:

```php	
// Disable the installer
$config['enable_installer'] = false;
```

```console
root@ubuntu# rm -rf /var/www/webmail/installer
```

## Test

Go to http://<span></span>webmail.myserver<span></span>.com/ and log in using a valid email user on your system (mailuser@<span></span>mailhostname.<span></span>com). Then try sending yourself an email message. Any errors will be logged to /var/www/webmail/logs/errors. Using this setup, Roundcube will authenticate through SASL and Dovecot in order to communicate with the wider internet.

## Problems

### The Sent/Junk/Trash folders don't do anything!

Roundcube calls these "special folders" and you can configure them under the Settings -> Special Folders area. Ensure the "Draft", "Sent", and "Junk" folders are set to the appropriate locations in your email account.

### I can't send large file attachments!

Edit /etc/php/7.0/fpm/php.ini to set file sizes to something reasonable, like 5 MB. I wouldn't make these too big. Note that this will affect all PHP-based systems on your server.
	
        upload_max_filesize = 5M
        post_max_size = 5M

Since Nginx doesn't use Roundcube's .htaccess files, their rcube_utils::max_upload_size() setting will be ignored. You may also need to set a max upload size in the Nginx config files, for example in /etc/nginx/nginx.conf:
	
        client_max_body_size 5M;
        
### Roundcube doesn't filter spam!

You could set up Sieve to filter junk mail automatically. But since we already configured Spamassassin to tag identified spam as ***** SPAM *****, we have an easier solution. All we need to do is ensure that this ends up in the spam folder -- much like our desktop or phone email clients can be set to do. So grab the filters plugin:

```console	
root@ubuntu# cd /var/www/webmail/plugins
root@ubuntu# git clone https://github.com/6ec123321/filters.git
```

In webmail/config/config.inc.php, enable the 'filters' plugin:

```php
// List of active plugins (in plugins/ directory)
$config['plugins'] = array('archive', 'filters', 'zipdownload');
```

Now, back in Roundcube, go to your Settings-> Filters button and add a filter to stick any message whose subject contains "***** SPAM *****" in your junk folder.

### I can't change my password!

Back in webmail/config/config.inc.php, add the 'password' plugin:

```php
// List of active plugins (in plugins/ directory)
$config['plugins'] = array('archive', 'filters', 'password', 'zipdownload');
```

Next, copy /var/www/webmail/plugins/password/config.inc.php.dist to config.inc.php and tell it to use the SQL database and query we created in our original server setup for adding a user password.

```php
// These from our Postfix-Dovecot mailserver article
$config['password_db_dsn'] = 'mysql://mailuser:password@127.0.0.1/mailserver';

$config['password_query'] = 'UPDATE virtual_users SET password = ENCRYPT(%p, CONCAT("$6$", SUBSTRING(SHA(RAND()), -16))) WHERE email = %u';
```

(Note: You may need to give the UPDATE privilege to mailuser if you haven't already.)

Now if you go to Settings in Roundcube you'll have a password change button where you can enter your current and new passwords.