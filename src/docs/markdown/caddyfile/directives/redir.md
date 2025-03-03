---
title: redir (Caddyfile directive)
---

# redir

Issues an HTTP redirect to the client.

This directive implies that a matched request is to be rejected. It is ordered very early in the [handler chain](/docs/caddyfile/directives#directive-order) (before [`rewrite`](/docs/caddyfile/directives/rewrite)).


## Syntax

```caddy-d
redir [<matcher>] <to> [<code>]
```

- **&lt;to&gt;** is the target location. Becomes the response's Location header.
- **&lt;code&gt;** is the HTTP status code to use for the redirect. Can be:
	- A positive integer in the 3xx range, or 401
	- `temporary` for a temporary redirect (302; default)
	- `permanent` for a permanent redirect (301)
	- `html` to use an HTML document to perform the redirect (useful for redirecting browsers but not API clients)
	- A placeholder with a status code value



## Examples

Redirect all requests to `https://example.com`:

```caddy-d
redir https://example.com
```

Same, but preserve the existing URI by appending the [`{uri}` placeholder](/docs/caddyfile/concepts#placeholders):

```caddy-d
redir https://example.com{uri}
```

Same, but permanent:

```caddy-d
redir https://example.com{uri} permanent
```

Redirect your old `/about-us` page to your new `/about` page:

```caddy-d
redir /about-us /about
```
