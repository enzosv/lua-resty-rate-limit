## Changes from original version

This fork allows for additional operations before calling `ngx.exit`.
Note that it now becomes the developer's responsibility to perform `ngx.exit` before proceeding with the request.

## OpenResty Redis Backed Rate Limiter

This is a OpenResty Lua and Redis powered rate limiter. You can specify the number of requests to allow within a certain timespan, ie. 40 requests within 10 seconds. With this setting (as an example), you can burst to 40 requests in a single second if you wanted, but would have to wait 9 more seconds before being allowed to issue another.

One of the key reasons we built this was to be able to share the rate limit across our entire API fleet as opposed to individually on each instance. We've tested this to be stable with a single Redis instance processing over 20,000 requests per second.

lua-resty-rate-limit is considered production ready and is currently being used to power our rate limiting at [The Movie Database (TMDb)](https://www.themoviedb.org).

### OpenResty Prerequisite

You have to compile OpenResty with the `--with-http_realip_module` option.

### Needed in your nginx.conf

```
http {
    # http://serverfault.com/questions/331531/nginx-set-real-ip-from-aws-elb-load-balancer-address
    # http://serverfault.com/questions/331697/ip-range-for-internal-private-ip-of-amazon-elb
    set_real_ip_from            127.0.0.1;
    set_real_ip_from            10.0.0.0/8;
    set_real_ip_from            172.16.0.0/12;
    set_real_ip_from            192.168.0.0/16;
    real_ip_header              X-Forwarded-For;
    real_ip_recursive           on;
}
```

### Example OpenResty Site Config

```
# Location of this Lua package
lua_package_path "/opt/lua-resty-rate-limit/lib/?.lua;;";

upstream api {
    server unix:/run/api.sock;
}

server {
    listen 80;
    server_name api.dev;

    access_log  /var/log/openresty/api_access.log;
    error_log   /var/log/openresty/api_error.log;

    location / {
        access_by_lua '
            local request = require "resty.rate.limit"
            local limited = request.limit { key = ngx.var.remote_addr,
                            rate = 40,
                            interval = 10,
                            log_level = ngx.NOTICE,
                            redis_config = { host = "127.0.0.1", port = 6379, timeout = 1, pool_size = 100 },
                            whitelisted_api_keys = { ["XXX"] = true, ["ZZZ"] = true } }
            if limited then
                -- perform pre exit functions here such as logging
                ngx.exit(ngx.HTTP_OK)
            end
        ';

        proxy_set_header  Host               $host;
        proxy_set_header  X-Server-Scheme    $scheme;
        proxy_set_header  X-Real-IP          $remote_addr;
        proxy_set_header  X-Forwarded-For    $remote_addr;
        proxy_set_header  X-Forwarded-Proto  $x_forwarded_proto;

        proxy_connect_timeout  1s;
        proxy_read_timeout     30s;

        proxy_pass   http://api;
    }
}
```

### Config Values

You can customize the rate limiting options by changing the following values:

- key: The value to use as a unique identifier in Redis
- rate: The number of requests to allow within the specified interval
- interval: The number of seconds before the bucket expires
- log_level: Set an Nginx log level. All errors from this plugin will be dumped here
- redis_config: The Redis host, port, timeout and pool size
- whitelisted_api_keys: A lua table of API keys to skip the rate limit checks for

### License

MIT License

Copyright (c) 2016 Travis Bell

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
