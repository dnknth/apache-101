---
title: "Security"
date: 2018-05-30T20:40:55+02:00
---

The [first part](basic-apache-configuration.md) of the series covered the basic Apache configuration. Before installing any sites, let's go over the default settings and tighten them up.

As a rule of thumb,  everything is disabled / forbidden by default, and access is selectively re-enabled on a directory / virtual host basis. Let's get started.

# Server security

Edit `conf-enabled/security.conf`. The most important setting is the following, it affects all sites, including virtual ones:

```apache
    # Disable access to the entire file system except for the directories that
    # are explicitly allowed later.
    #
    # This currently breaks the configurations that come with some web application
    # Debian packages.
    <Directory />
       AllowOverride None
       Require all denied
    </Directory>
```

You can always fix access right issues later on (you *do* test your apps,don't you?). In case of doubt, check the error log.

Not really relevant to security, but who needs to know what exactly is installed?

```apache
     ServerTokens Prod
     ServerSignature Off
     TraceEnable Off
```

If you use 3rd party software from the interwebs, you probably also want:

```apache
    # If you use version control systems in your document root, you should
    # probably deny access to their directories. For example, for subversion:
    <DirectoryMatch "/\.svn">
       Require all denied
    </DirectoryMatch>

    <DirectoryMatch "/\.git">
       Require all denied
    </DirectoryMatch>
```

The following is paranoid, but according to [Andy Grove](https://en.wikipedia.org/wiki/Andrew_Grove), the paranoid die last. At any rate, it is recommended by [Mozilla](https://observatory.mozilla.org):

```apache
    # Setting this header will prevent other sites from embedding pages from this
    # site as frames. This defends against clickjacking attacks.
    # Requires mod_headers to be enabled.
    Header set X-Frame-Options: "sameorigin"
```

This is why `mod_headers` was part of the [basic settings](basic-apache-configuration.md).

# TLS Encryption

This [day and age](https://www.theguardian.com/us-news/the-nsa-files), most websites use encryption, because your personal blog is none of [NSA's](https://www.nsa.gov) or [GCHQ's](https://www.gchq.gov.uk) business. Also, HTTPS support will give you a better ranking in search engines.

## Ciphers

Let's set up some reasonable defaults for TLS: Head over to Mozilla's [SSL Configuration Generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/) to generate ciphers, using the 'modern' profile. Put them in `/etc/apache2/options/ssl-modern.conf`. At the time of writing, you'll get a straight `A+` grade on [SSLLabs](https://www.ssllabs.com/ssltest/) and [CryptCheck](https://tls.imirhil.fr) with:

```apache
    <IfModule mod_ssl.c>
        SSLEngine On

        SSLVerifyClient none
        SSLStrictSNIVHostCheck off

        # modern configuration, tweak to your needs
        # https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=apache-2.4.18&openssl=1.0.1f&hsts=yes&profile=modern
        SSLProtocol             ALL -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
        SSLCipherSuite          ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256

        SSLHonorCipherOrder     on
        SSLCompression          off
        SSLSessionTickets       off

        <FilesMatch "\.(cgi|shtml|phtml|php)$">
            SSLOptions +StdEnvVars
        </FilesMatch>
    </IfModule>
```

It is safe to use different cipher suites in different virtual hosts, even with SNI.

No cipher will last forever, better check them occasionally.

## Certificates

You need a certificate for each site name. The fastest and cheapest way for basic certs is [LetsEncrypt](https://letsencrypt.org). These are sufficient for transport-level security with domain validation, but lack the more official-looking organization and extended validation levels. Install the `certbot` tool:

    sudo apt-get install certbot
    
If you are a first-time Letsencrypt user, start off with:

    sudo certbot register

Add this to `/etc/apache2/conf-available/site.conf`, you'll need it in a minute for `certbot`'s `--webroot` plugin:

```apache
    # Use the same .well-known directory for all virtual hosts
    Alias /.well-known /var/www/.well-known
```

The following goes into `.bashrc` or `.bash_aliases`:

    alias letsencrypt='sudo certbot --webroot -w /var/www' --rsa-key-size 4096

Assuming that DNS names for `www.example.org` and `example.org` are already set up and point to our IP, you can install a cert (into `/etc/letsencrypt/live/example.org`) with:

    letsencrypt certonly -d example.org -d www.example.org

The corresponding TLS configuration in `/etc/apache2/sites-available/example.org` looks like this (logging and other stuff omitted):

```apache
    <VirtualHost *:443>

        ServerName example.org
        ServerAlias www.example.org
        ServerAdmin webmaster@example.org
        DocumentRoot /var/www
    
        Include options/ssl-modern.conf

        <IfModule mod_ssl.c>
            SSLCertificateFile    /etc/letsencrypt/live/example.org/cert.pem
            SSLCertificateKeyFile /etc/letsencrypt/live/example.org/privkey.pem
            SSLCACertificateFile  /etc/letsencrypt/live/example.org/chain.pem
        </IfModule>

    </VirtualHost>
```

## OCSP Stapling

With [OSCP stapling](http://httpd.apache.org/docs/2.4/ssl/ssl_howto.html#ocspstapling) the certificate revocation status is sent along during the TLS handshake. This improves security and speeds up the initial connection.
Add the following to `conf-enabled/oscp-stapling.conf` (don't forget to enable):

```apache
    # OCSP Stapling, only in httpd 2.3.3 and later
    <IfModule mod_ssl.c>
    	SSLUseStapling          on
    	SSLStaplingResponderTimeout 5
    	SSLStaplingReturnResponderErrors off
    	SSLStaplingCache        shmcb:/var/run/ocsp(128000)
    </IfModule>
```
    
Test it with:
    
    openssl s_client -connect example.org:443 -status -servername example.org
    
and look for:

    OCSP response: 
    ======================================
    OCSP Response Data:
        OCSP Response Status: successful (0x0)
        Response Type: Basic OCSP Response
    ...
        Cert Status: Good

# Strict Transport Security (HSTS)

Mixing HTTP and HTTPS is not a good idea. Browsers will complain, and pages may not load completely. Therefore, let's aim for HTTPS **only**, by adding 2 things to the web server configuration:

1. Redirect all HTTP requests to HTTPS
2. Add response headers telling browsers to always use TLS

## Redirect to HTTPS

To force HTTPs, add a snippet `options/https-redirect.conf`:

```apache
    # Redirect to HTTPS
    DocumentRoot /var/www

    <IfModule mod_rewrite.c>
        RewriteEngine On
        # RewriteCond %{HTTPS} off
        RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
    </IfModule>
```
    
Then add to `/etc/apache2/sites-available/example.org`:

```apache
    <VirtualHost *:80>
        ServerName   example.org
        ServerAlias  www.example.org
        ServerAdmin  webmaster@example.org
        Include      options/https-redirect.conf
    </VirtualHost>
```

## HSTS response headers

The next snippet will be included in `<VirtualHost *:443>` sections. Lets's put it in `options/hsts.conf`:
    
```apache
    <IfModule mod_headers.c>
       Header always set Strict-Transport-Security "max-age=15768000; includeSubDomains"
       
       # Always ensure Cookies have "Secure" set (JAH 2012/1)
       Header edit Set-Cookie (?i)^(.*)(;\s*secure)??((\s*;)?(.*)) "$1; Secure$3$4"
    </IfModule>
```

HSTS is not easily revokable, because browsers that have seen the header will use HTTPS exclusively for the duration of `max-age`. Enable it only if you are positive that HTTP (without encryption) is not needed.

# Content security

Modern browsers honour [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy) headers sent by the web server. That's great, because it lets you declare *exactly* what's allowed for your web application, and mitigates all sorts of [XSS](https://en.wikipedia.org/wiki/Cross-site_scripting) and frame hijacking attacks. 

As a secure baseline, add this is `options/csp-strict.conf`:

```apache
    Header setifempty Content-Security-Policy "\
        default-src 'none';\
        base-uri 'none';\
        form-action 'self';\
        frame-ancestors 'self';\
        connect-src 'self';\
        font-src 'self';\
        frame-src 'self';\
        img-src 'self';\
        media-src 'self';\
        object-src 'self';\
        script-src 'self';\
        style-src 'self';\
    "
```
    
It ensures that all resources are loaded from your site. Include it in your virtual hosts like so:

```apache
    Include options/csp-strict.conf
```

Of course, there are always exceptions, especially if you run 3rd party apps with special needs like inline styles and Javascript. For these, put into `options/csp-lenient.conf` (adjust to taste):

```apache
    Header set Content-Security-Policy "\
        default-src 'none';\
        base-uri 'none';\
        form-action 'self';\
        frame-ancestors 'self';\
        child-src 'self';\
        connect-src 'self';\
        font-src 'self';\
        frame-src 'self';\
        img-src data: 'self';\
        media-src 'self';\
        object-src 'self';\
        script-src 'self' 'unsafe-inline' 'unsafe-eval';\
        style-src 'self' 'unsafe-inline';\
    "
```
    
Then you can override the secure default with the relaxed option on a directory like so:

```apache
    <Directory /usr/local/share/my/unsafe/app>
        Include options/csp-lenient.conf
    </Directory>
```

The [next post](static-resources.md) explains how to serve static files at top speed.
