---
title: "Static Resources"
date: 2018-05-31T18:44:32+02:00
---

Any run-of-the mill web site today uses a busload of static resources, typically including:

* [JQuery](http://jquery.com)
* [Bootstrap](https://getbootstrap.com)
* A few web fonts, maybe from [Google](https://fonts.google.com), or [FontAwesome](https://fontawesome.com)

In a production environment, static resources will be served through a CDN or dedicated low-latency servers. For small to medium-sized sites, this distinction is rarely made, but there are good reasons to offload static files:

* They pollute your access logs,
* Page load times increase noticably if you just dump static into a directory. 

In short, Apache is not the best solution for static files, and you will do yourself a favour if you avoid the problem via:

* A CDN,
* Serving static from an AWS S3 bucket,
* Offloading static to [nginx](https://nginx.org) on another machine.

If you absolutely must serve static, let's say in a low-traffic site or test environment, then read on.

# Logging

In a production environment, logging static content provides almost no useful additional information. On the contrary, it makes it difficult to see (and analyse) relevant page hits. To solve this, let's add a snippet in `/etc/apache2/options/tag-static.conf`:

```apache
    <IfModule mod_headers.c>
        SetEnvIfNoCase Request_URI "\.(?:css|gif|jpe?g|js|png|svg|ttf|woff2?)$" nolog
    </IfModule>
```
    
This tags static via a request variable. You can then include it into virtual hosts like so:

```apache
    Include options/tag-static.conf
```

and selectively log with:

```apache
    CustomLog ${APACHE_LOG_DIR}/access.log combined env=!nolog
```

# Page load times

Let's bring the page load time down in three steps:

1. Extract static content into separate directories,
2. Instruct the browser and intermediate proxies to cache it,
3. Serve compressed files.

## Extract static files

This one really depends on your web applications. Good web frameworks already [collect](https://docs.djangoproject.com/en/2.0/ref/contrib/staticfiles/) static for easy deployment. Or if you use [Bower](https://bower.io), everything in `bower_components` is static.

For the sake of this tutorial, let's assume you worry about European GDPR and want to self-host some Google webfonts. They are already [installed](https://google-webfonts-helper.herokuapp.com/fonts) in `/usr/local/share/fonts/google`.

## Enable caching

Unless an [`Expires`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Expires) HTTP header is present, the browser will check on *every request* if a resource is still up-to-date. This is pointless for static files because they never change. For the web font scenario, add some `mod_expires` best-before dates:

```apache
    <Directory /usr/local/share/fonts>
        Require all granted
        AddType font/woff2 .woff2

    	<IfModule mod_expires.c>
    		ExpiresActive On
    		ExpiresByType text/css "access plus 1 month"
    		ExpiresByType application/font-sfnt "access plus 1 year"
    		ExpiresByType application/font-woff "access plus 1 year"
    		ExpiresByType font/woff2 "access plus 1 year"
    		ExpiresByType application/vnd.ms-fontobject "access plus 1 year"
    		ExpiresByType image/svg+xml "access plus 1 year"
    	</IfModule>
    </Directory>
```

This means that browsers will attempt to cache stuff for the maximum recommended time, which is one year. In reality, they will come back more often, but certainly not every day.

For other static Javascript / CSS / image resources, use something like this:

```apache
    <IfModule mod_expires.c>
        ExpiresActive On
        ExpiresByType text/css "access plus 1 month"
        ExpiresByType application/xslt+xml "access plus 1 month"
        ExpiresByType text/javascript "access plus 1 month"
        ExpiresByType application/javascript "access plus 1 month"
        ExpiresByType application/x-javascript "access plus 1 month"
        ExpiresByType image/gif "access plus 1 month"
        ExpiresByType image/jpeg "access plus 1 month"
        ExpiresByType image/png "access plus 1 month"
        ExpiresByType image/x-icon "access plus 1 month"
        ExpiresByType image/svg+xml "access plus 1 month"
        ExpiresByType image/vnd.microsoft.icon "access plus 1 month"
    </IfModule>
```

You can verify in your browser's developer tools (in the 'Network' tab) that things are indeed cached and not transferred on page reload.

## Compression

You probably have `mod_deflate` installed as [recommended](basic-apache-configuration.md). For static files, that is *not* the way to go, as it will compress files on *every* request. With dozens of files to download and compress for the initial page load, the CPU times for compression add up as extra latency.

Most browsers support `gzip`, `deflate` and `br` transfer encodings. The way to go is to *manually* compress all static files on disk, and add a bit of [content negotiation](http://httpd.apache.org/docs/2.4/en/mod/mod_negotiation.md) configuration to deliver the smallest possible alternative. This only takes a few milliseconds per request.

For manual pre-compression, first get a modern compressor:

    sudo apt-get install brotli

Brotli compression is already used in `woff`, `woff2` and `eot` web fonts, so do not bother to compress them further. For the repetitive work, add a little shell script in `/usr/local/bin/precompress` and make it executable:

```shell
    #!/bin/sh

    set -e
    umask 0022

    die() {
        echo $*
        exit 1
    }

    for f ; do
    	ext="${f##*.}"
    	case "$ext" in
    		# Skip internally compressed formats
    	    brotli|eot|gz|png|woff*|zip) continue ;;
        esac

    	if [ -f "$f.$ext" ] ; then
    		mv "$f.$ext" "$f"
    		rm -f "$f.brotli" "$f.gz"
        elif [ -f "$f" ] ; then
    		gzip -k "$f"
    		if [ -x /usr/bin/brotli ] ; then
    			brotli --input "$f" --output "$f.brotli"
    			chmod +r "$f.brotli"
    		fi
    		mv "$f" "$f.$ext"
    	else
    		die "$f.$ext" not found
        fi
    done
```

This will work on Debian `stretch`. Recent versions of `brotli` use different options, and you'll need to adjust line 24 as per the [manual page](http://manpages.ubuntu.com/manpages/bionic/en/man1/brotli.1.md):

```shell
	brotli -S .brotli -k "$f"
```

Then head to `/usr/local/share/fonts/google` and 

    precompress *
    
That was not too hard, was it? Notice that compression takes a while, and imagine part of that happening on each request. By the way, the script works both ways. To decompress, just call is again without the added file extensions like so:

    precompress roboto-v18-latin-100italic.svg

As for the web server configuration, even the [official documentation](https://httpd.apache.org/docs/2.4/mod/mod_deflate.html#precompressed) on the topic proposes using `mod_rewrite`. This is pretty horrid, because it does not honour the browser's preferred order of encodings.

A better solution is based on [multiviews](https://kevinlocke.name/bits/2016/01/20/serving-pre-compressed-files-with-apache-multiviews/). Create `options/precompressed.conf`:

```apache
    <IfModule mod_negotiation.c>
        # Enable MultiViews for content negotiation
    	Options +MultiViews

        # Treat .gz as gzip encoding, not application/gzip type
        RemoveType .gz
        AddEncoding gzip .gz
        AddEncoding br   .brotli
    </IfModule>
```

and use it like so:

```apache
    <Directory /usr/local/share/fonts>
        Include options/precompressed.conf
    </Directory>
```
    
To test, check your browser's *Network* tab under *Developer tools*. For each compressed resource, the `Content-Location` response header reveals the negotiated variant.

## Fallback options

If you must host an application whose sloppy programmers who did not bother to extract static files properly, pre-compression is not an option. For example, Wordpress ships with a truckload of plugins, each of which may bring its own Javascript files, scattered in dozens of directories.

If you are runinng high-volume sites with crappy applications like this, you are out of luck, because that will throttle down your web servers and waste precious bandwidth for no effect. Go to management and ask for more OpEx.

For sites with moderate traffic, you still have options, even if they are far from ideal. It boils down to trade CPU cycles (for on-the-fly compression) against wasting bandwith to dish out static. Initially, compression will boost page load times, but only until the CPUs get clogged. At any rate, page load times *won't* be optimal.

Create `/etc/apache2/conf-available/deflate.conf` like this (and don't forget to enable it):

```apache
    <IfModule mod_deflate.c>
        <IfModule mod_filter.c>
            AddOutputFilterByType DEFLATE text/html text/plain text/xml
            AddOutputFilterByType DEFLATE application/x-javascript application/javascript application/ecmascript
            AddOutputFilterByType DEFLATE text/css application/json
            AddOutputFilterByType DEFLATE application/rss+xml application/svg+xml application/xml
        </IfModule>
    </IfModule>
```

You can include this in any virtual host, `<Directory>` or `<Location>` sections of your site. Just make sure that pre-compression and on-the-fly compression are not enabled both at once, because then `mod_deflate` will always kick in.

The [next post](pretty-directory-listings.md) will wrap things up with eye candy for directory listings.
