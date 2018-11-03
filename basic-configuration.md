---
title: "Basic Apache configuration"
date: 2018-05-30T16:02:04+02:00
---

This first post explains the very basics, namely the configuration directory structure and required modules. I mainly use Linux and Mac OS, so the examples should work on most flavours of Unix. On Windows with XAMPP, your mileage may vary. 

# Installation

Let's assume you've just installed Apache from your favourite Linux distro, e.g. with 

    sudo apt-get install apache2

At the time of writing, the current version is 2.4, and that's what you should be using. On Debian flavours, you'll end up with a directory structure like this:

    /etc/apache2/
    ├── conf-available
    ├── conf-enabled
    ├── mods-available
    ├── mods-enabled
    ├── sites-available
    └── sites-enabled

Also, the commands `a2(en|dis)(conf|mod|site)` can be found in `/usr/sbin`.

I ususally name the configuration file for each site exactly like the (main) DNS name, so `example.org` would end up in `/etc/apache2/sites-available/example.org`. This does not agree with Debian distro defaults, but let's change this line near the end of `/etc/apache2/apache2.conf` from

```apache
    IncludeOptional sites-enabled/*.conf
```

to 

```apache
    IncludeOptional sites-enabled/*
```

In almost every case, you'll end up with more than one virtual host. Configurations tend to have repetitive sections, and it will pay off pretty soon if you collect them in one place:

    sudo mkdir /etc/apache2/options
    
Why `options` instead of `conf-available`? Snippets in `conf-available` are pulled into the **main** server configuration (via links in `conf-enabled`), but our stuff will be included in **virtual host** configs.

Next comes some global (default) configuration for all virtual hosts and the main server. It used to go in `/etc/apache2/httpd.conf`, but at least in Debian this is not present any longer. So, let's put it in `/etc/apache2/conf-available/site.conf` (do not forget to enable it with `a2enconf site`). The initial content is simple enough, you'll add more a bit later:

```apache
    ServerName localhost
    ServerAdmin webmaster@example.org
    BufferedLogs On
    LogLevel error
```

# Modules

First and foremost, let's get rid of `mod_access_compat`. It provides 2.2-compatible access control, and that was a bit of a mess. If you later happen to install an application that still relies on 2.2-style settings, please just fix it manually, it's not complicated.

    sudo a2dismod access_compat

Now, let's clean up in `mods-enabled`. Some distros install way too much there. Who still uses CGI today?

These ones are useful for the basic setup:

* alias
* auth_basic
* autoindex
* deflate
* dir
* env
* expires
* filter
* headers
* mime
* mpm_prefork
* negotiation
* setenvif
* socache_shmcb
* ssl

You can always install more as needed.

# Ports

Fot HTTPS, the server needs to listen on ports 80 and 443. On Debian, this is done in `/etc/apache2/ports.conf`:

```apache
    Listen 80

    <IfModule ssl_module>
    	Listen 443
    </IfModule>

    <IfModule mod_gnutls.c>
    	Listen 443
    </IfModule>
```
    
Enabling SSL is covered in the [next article](security.md).

# Logging

## Log files

With the default configuration, Apache will log to `/var/log/apache2/(access|error).log`, which are rotated on a weekly basis for about one year. This does not make much sense at all. The access log can get *huge* in a week, and errors that are older than a few weeks should have been fixed by now.

So, let's tune settings in `/etc/logrotate.d/apache2`. I use the following values, just alter to taste:

    /var/log/apache2/*.log {
        daily
        missingok
        rotate 14
        compress
        delaycompress
        notifempty
        # more stuff ....
    }

Next, have a look at `/etc/apache2/conf-available/other-vhosts-access-log.conf`. It says:

```apache
    # Define an access log for VirtualHosts that don't define their own logfile
    CustomLog ${APACHE_LOG_DIR}/other_vhosts_access.log vhost_combined
```

Virtual host logging is pretty useless e.g. for redirection sites, so  the way to go is:

    a2disconf other-vhosts-access-log

## Access log format

Several log formats are usually [preconfigured](https://httpd.apache.org/docs/2.4/logs.html#common) in `/etc/apache2/apache2.conf`, such as `common`, `combined` etc. Most people will go for the `combined` format, as it provides most details for analysis.

This is fine *unless* your server relies on a reverse proxy for SSL termination. In that case, you'd end up logging the IP of the **proxy** server instead of the client IP. To get the correct (client) IP, add to `options/site.conf`:

```apache
    LogFormat "%{X-Forwarded-For}i %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined_proxy
```
    
## Access log content

To selectively disable logging on a directory, location or request method, this snippet goes in `/etc/apache2/options/nolog.conf`:

```apache
    <IfModule mod_headers.c>
        SetEnv nolog
    </IfModule>
```

Use it like so, for example to exclude static resources ([more on this later](static-resources.md)):

```apache
    CustomLog ${APACHE_LOG_DIR}/access.log combined env=!nolog
    
    <Directory "/usr/local/share/fonts/">
        Include options/nolog.conf
    </Directory>
```
    
# Default character set

In 1993, when [Tim B-L](https://en.wikipedia.org/wiki/Tim_Berners-Lee) (now Sir Tim, KBE etc.) invented the WWW, he was content with Europan script. So, ISO-8859-1 (*Latin-1*) became the de-facto standard, and as per HTTP specification, it still is today.

Today, we have [Unicode](https://en.wikipedia.org/wiki/Unicode), and UTF8 is a reasonable default pretty much everywhere outside of Asia. Add or edit in `/etc/apache2/conf-available/charset.conf `:

```apache
    AddDefaultCharset UTF-8
```

Technically, this non-standard, but empirically the best choice for modern websites using European languages. Make sure to enable it with

    a2enconf charset

# Wrap up

Let's have a final look into `/etc/apache2/conf-enabled`. As of now, it should have exactly 3 symlinks:

* charset.conf
* security.conf
* site.conf

Wipe everything else with `a2disconf`, or just manually delete the symlinks.

Now, you'll want to test any changes with 

    sudo apache2ctl configtest

and only then commit them with

    sudo apache2ctl graceful

I suggest to create aliases for these in your `.bashrc` or `.bash_aliases`, bacause you'll need them frequently. Mine are called `AC` and `AG`.

The [next post](security.md) will look into security and SSL.
