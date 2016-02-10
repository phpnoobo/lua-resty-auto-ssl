# lua-resty-auto-ssl

On the fly (and free) SSL registration and renewal inside [OpenResty/nginx](http://openresty.org) with [Let's Encrypt](https://letsencrypt.org).

This OpenResty plugin automatically and transparently issues SSL certificates from Let's Encrypt (a free certificate authority) as requests are received. It works like:

- A SSL request for a SNI hostname is received.
- If the system already has a SSL certificate for that domain, it is immediately returned (with OCSP stapling).
- If the system does not yet have an SSL certificate for this domain, it issues a new SSL certificate from Let's Encrypt. Domain validation is handled for you. After receiving the new certificate (usually within a few seconds), the new certificate is saved, cached, and returned to the client (without dropping the original request).

This uses the `ssl_certificate_by_lua` functionality in OpenResty 1.9.7.2+.

## Status

The primary functionality is in place, but there are still a few [todos](#todo) and cleanup that needs to be done before this is probably ready for any real use.

## Installation

Requirements:

- [OpenResty](http://openresty.org/#Download) 1.9.7.2 or higher
- OpenSSL 1.0.2e or higher
- [LuaRocks](http://openresty.org/#UsingLuaRocks)

```sh
$ sudo luarocks install https://raw.githubusercontent.com/GUI/lua-resty-auto-ssl/master/lua-resty-auto-ssl-git-1.rockspec

# Create /etc/resty-auto-ssl and make sure it's writable by whichever user your
# nginx workers run as (in this example, "www-data").
$ sudo mkdir /etc/resty-auto-ssl
$ sudo chown www-data /etc/resty-auto-ssl
```

Implement the necessary configuration inside your nginx config. Here is a minimal example:

```nginx
events {
  worker_connections 1024;
}

http {
  # The "auto_ssl" shared dict must be defined with enough storage space to
  # hold your certificate data.
  lua_shared_dict auto_ssl 1m;

  # A DNS resolver must be defined for OSCP stapling to function.
  resolver 8.8.8.8;

  # Intial setup tasks.
  init_worker_by_lua_block {
    auto_ssl = (require "resty.auto-ssl").new()

    -- Define a function to determine which SNI domains to automatically handle
    -- and register new certificates for. Defaults to not allowing any domains,
    -- so this must be configured.
    auto_ssl:set("allow_domain", function(domain)
      return true
    end)

    auto_ssl:init_worker()
  }

  # HTTPS server
  server {
    listen 443 ssl;

    # Dynamic handler for issuing or returning certs for SNI domains.
    ssl_certificate_by_lua_block {
      auto_ssl:ssl_certificate()
    }

    # You must still define a static ssl_certificate file for nginx to start.
    #
    # You may generate a self-signed fallback with:
    #
    # openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 \
    #   -subj '/CN=resty-auto-ssl-fallback' \
    #   -keyout /etc/ssl/resty-auto-ssl-fallback.key \
    #   -out /etc/ssl/resty-auto-ssl-fallback.crt
    ssl_certificate /etc/ssl/resty-auto-ssl-fallback.crt;
    ssl_certificate_key /etc/ssl/resty-auto-ssl-fallback.key;
  }

  # HTTP server
  server {
    listen 80;

    # Endpoint used for performing domain verification with Let's Encrypt.
    location /.well-known/acme-challenge/ {
      content_by_lua_block {
        auto_ssl:challenge_server()
      }
    }
  }

  # Internal server running on port 8999 for handling certificate tasks.
  server {
    listen 127.0.0.1:8999;
    location / {
      content_by_lua_block {
        auto_ssl:hook_server()
      }
    }
  }
}
```

## Configuration

Additional configuration options can be set on the `auto_ssl` instance that is created:

- **`allow_domain`**
  *Default:* `function(domain) return false end`

  A function that determines whether the incoming SNI domain should automatically issue a new SSL certificate.

  By default, resty-auto-ssl will not perform any SSL registrations until you define the `allow_domain` function. You may return `true` to handle all possible domains, but be aware that bogus SNI hostnames can then be used to trigger an indefinite number of SSL registration attempts (which will be rejected). A better approach may be to whitelist the allowed domains in some way.

  *Example:*

  ```lua
  auto_ssl:set("allow_domain", function(domain)
    return ngx.re.match(domain, "^(example.com|example.net)$", "ijo")
  })
  ```

- **`dir`**
  *Default:* `/etc/resty-auto-ssl`

  The base directory used for storing configuration, temporary files, and certificate files (if using the `file` storage adapter). This directory must be writable by the user nginx workers run as.

  *Example:*

  ```lua
  auto_ssl:set("dir", "/some/other/location")
  ```

- **`storage_adapter`**
  *Default:* `resty.auto-ssl.storage_adapters.file`
  *Options:* `resty.auto-ssl.storage_adapters.file`, `resty.auto-ssl.storage_adapters.redis`

  The storage mechanism used for persistent storage of the SSL certificates. File-based and redis-based storage adapters are supplied, but custom external adapters may also be specified (the value simply needs to be on the `lua_package_path`).

  The default storage adapter persists the certificates to local files. However, you may want to consider another storage adapter (like redis) for a couple reason:
    - File I/O causes blocking in OpenResty which should be avoided for optimal performance. However, files are only read and written the first time a certificate is seen, and then things are cached in memory, so the actual amount of file I/O should be quite minimal.
    - Local files won't work if the certificates need to be shared across multiple servers (for a load-balanced environment).

  *Example:*

  ```lua
  auto_ssl:set("storage_adapter", "resty.auto-ssl.storage_adapters.redis")
  ```

- **`redis`**
  *Default:* `{ host = "127.0.0.1", port = 6379 }`

  If the `redis` storage adapter is being used, then additional connection options can be specified on this table. Accepts `host`, `port`, and `socket` (for unix socket paths) options.

  *Example:*

  ```lua
  auto_ssl:set("redis", {
    host = "10.10.10.1"
  })
  ```

## Precautions

- **Allowed Hosts:** By default, resty-auto-ssl will not perform any SSL registrations until you define the `allow_domain` function. You may return `true` to handle all possible domains, but be aware that bogus SNI hostnames can then be used to trigger an indefinite number of SSL registration attempts (which will be rejected). A better approach may be to whitelist the allowed domains in some way.
- **Untrusted Code:** Ensure your OpenResty server where this is installed cannot execute untrusted code. The certificates and private keys have to be readable by the web server user, so it's important that this data is not compromised.
- **File Storage:** The default storage adapter persists the certificates to local files. However, you may want to consider another storage adapter (like redis) for a couple reason:
  - File I/O causes blocking in OpenResty which should be avoided for optimal performance. However, files are only read and written the first time a certificate is seen, and then things are cached in memory, so the actual amount of file I/O should be quite minimal.
  - Local files won't work if the certificates need to be shared across multiple servers (for a load-balanced environment).

## TODO

- Implement background task to perform automatic renewals.