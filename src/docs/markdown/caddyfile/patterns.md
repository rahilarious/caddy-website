---
title: Common Caddyfile Patterns
---

# Common Caddyfile Patterns

This page demonstrates a few complete and minimal Caddyfile configurations for common use cases. These can be helpful starting points for your own Caddyfile documents.

These are not drop-in solutions; you will have to customize your domain name, ports/sockets, directory paths, etc. They are intended to illustrate some of the most common configuration patterns.

- [Static file server](#static-file-server)
- [Reverse proxy](#reverse-proxy)
- [PHP](#php)
- [Redirect `www.` subdomain](#redirect-www-subdomain)
- [Trailing slashes](#trailing-slashes)
- [Wildcard certificates](#wildcard-certificates)
- [Single-page apps (SPAs)](#single-page-apps-spas)


## Static file server

```caddy
example.com {
	root * /var/www
	file_server
}
```

As usual, the first line is the site address. The [`root` directive](/docs/caddyfile/directives/root) specifies the path to the root of the site (the `*` means to match all requests, so as to disambiguate from a [path matcher](/docs/caddyfile/matchers#path-matchers))&mdash;change the path to your site if it isn't the current working directory. Finally, we enable the [static file server](/docs/caddyfile/directives/file_server).



## Reverse proxy

Proxy all requests:

```caddy
example.com {
	reverse_proxy localhost:5000
}
```

Only proxy requests having a path starting with `/api/` and serve static files for everything else:

```caddy
example.com {
	root * /var/www
	reverse_proxy /api/* localhost:5000
	file_server
}
```

This uses a [request matcher](/docs/caddyfile/matchers#syntax) to match only requests that start with `/api/` and proxy them to the backend. All other requests will be served from the site [`root`](/docs/caddyfile/directives/root) with the [static file server](/docs/caddyfile/directives/file_server). This also depends on the fact that `reverse_proxy` is higher on the [directive order](/docs/caddyfile/directives#directive-order) than `file_server`.

There are many more [`reverse_proxy` examples here](/docs/caddyfile/directives/reverse_proxy#examples).



## PHP

With a PHP FastCGI service running, something like this works for most modern PHP apps:

```caddy
example.com {
	root * /srv/public
	encode gzip
	php_fastcgi localhost:9000
	file_server
}
```

Customize the site root accordingly; this example assumes that your PHP app's webroot is within a `public` directory&mdash;requests for files that exist on disk will be served with [`file_server`](/docs/caddyfile/directives/file_server), and anything else will be routed to `index.php` for handling by the PHP app.

You may sometimes use a unix socket to connect to PHP-FPM:

```caddy-d
php_fastcgi unix//run/php/php8.2-fpm.sock
```

The [`php_fastcgi` directive](/docs/caddyfile/directives/php_fastcgi) is actually just a shortcut for [several pieces of configuration](/docs/caddyfile/directives/php_fastcgi#expanded-form).



## Redirect `www.` subdomain

To **add** the `www.` subdomain with an HTTP redirect:

```caddy
example.com {
	redir https://www.{host}{uri}
}

www.example.com {
}
```


To **remove** it:

```caddy
www.example.com {
	redir https://example.com{uri}
}

example.com {
}
```


To remove it for **multiple domains** at once; this uses the `{labels.*}` placeholders which are the parts of the hostname, `0`-indexed from the right (e.g. `0`=`com`, `1`=`example-one`, `2`=`www`):

```caddy
www.example-one.com, www.example-two.com {
	redir https://{labels.1}.{labels.0}{uri}
}

example-one.com, example-two.com {
}
```



## Trailing slashes

You will not usually need to configure this yourself; the [`file_server` directive](/docs/caddyfile/directives/file_server) will automatically add or remove trailing slashes from requests by way of HTTP redirects, depending on whether the requested resource is a directory or file, respectively.

However, if you need to, you can still enforce trailing slashes with your config. There are two ways to do it: internally or externally.

### Internal enforcement

This uses the [`rewrite`](/docs/caddyfile/directives/rewrite) directive. Caddy will rewrite the URI internally to add or remove the trailing slash:

```caddy
example.com {
	rewrite /add     /add/
	rewrite /remove/ /remove
}
```

Using a rewrite, requests with and without the trailing slash will be the same.


### External enforcement

This uses the [`redir`](/docs/caddyfile/directives/redir) directive. Caddy will ask the browser to change the URI to add or remove the trailing slash:

```caddy
example.com {
	redir /add     /add/
	redir /remove/ /remove
}
```

Using a redirect, the client will have to re-issue the request, enforcing a single acceptable URI for a resource.



## Wildcard certificates

If you need to serve multiple subdomains with the same wildcard certificate, the best way to handle them is with a Caddyfile like this, making use of the [`handle` directive](/docs/caddyfile/directives/handle) and [`host` matchers](/docs/caddyfile/matchers#host):

```caddy
*.example.com {
	tls {
		dns <provider_name> [<params...>]
	}

	@foo host foo.example.com
	handle @foo {
		respond "Foo!"
	}

	@bar host bar.example.com
	handle @bar {
		respond "Bar!"
	}

	# Fallback for otherwise unhandled domains
	handle {
		abort
	}
}
```

You must enable the [ACME DNS challenge](/docs/automatic-https#dns-challenge) to have Caddy automatically manage wildcard certificates.



## Single-page apps (SPAs)

When a web page does its own routing, servers may receive lots of requests for pages that don't exist server-side, but which are renderable client-side as long as the singular index file is served instead. Web applications architected like this are known as SPAs, or single-page apps.

The main idea is to have the server "try files" to see if the requested file exists server-side, and if not, fall back to an index file where the client does the routing (usually with client-side JavaScript).

A typical SPA config usually looks something like this:

```caddy
example.com {
	root * /path/to/site
	encode gzip
	try_files {path} /index.html
	file_server
}
```

If your SPA is coupled with an API or other server-side-only endpoints, you will want to use `handle` blocks to treat them exclusively:

```caddy
example.com {
	encode gzip

	handle /api/* {
		reverse_proxy backend:8000
	}

	handle {
		root * /path/to/site
		try_files {path} /index.html
		file_server
	}
}
```
