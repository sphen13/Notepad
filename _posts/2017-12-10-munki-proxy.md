---
layout: post
title: "Munki Proxy - My First Real Docker Project"
description:
headline:
modified: 2017-12-10
category: software
tags: [docker, munki, nginx]
imagefeature: munki-proxy.gif
mathjax:
chart:
comments: true
featured: true
---

So I have just recently discovered [Docker](https://www.docker.com).  I may be the last person in the world to know about it, but after playing around with it a bit and figuring things out, I think its one of the best things right now.  I wanted to take on a small project that I could utilize docker for and [munki-proxy](https://github.com/sphen13/munki-proxy) is what I came up with.

Updated Dec 30th. Post can be found [here](/software/munki-proxy-update)
{: .notice}

<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

## A Little Background

For those of you not familiar with Docker, it is basically a **container-based virtual environment**.  Well - maybe.  At least thats what I think of it.  Better put in someone else's words:

> Docker is a tool designed to make it easier to create, deploy, and run applications by using containers. Containers allow a developer to package up an application with all of the parts it needs, such as libraries and other dependencies, and ship it all out as one package. By doing so, thanks to the container, the developer can rest assured that the application will run on any other Linux machine regardless of any customized settings that machine might have that could differ from the machine used for writing and testing the code. [Source](https://opensource.com/resources/what-docker)

It is a great way to get a fully-deployable application to consistently give the same results independent of what type of platform or hardware you are running on.  As long as you can run Docker you should be able to deploy a container fairly easily.  The docker container is based on the idea of an **immutable image**.  This makes sure there is a consistent environment when launching a container and allows you to update the image of a container very easily without the risk of losing data _(if you do things right)_.

I was mainly introduced to Docker by way of Graham Gilbert and his [blog](https://grahamgilbert.com).  He has an excellent [Crypt](https://github.com/grahamgilbert/crypt) project which has a docker based [Crypt-Server](https://github.com/grahamgilbert/Crypt-Server) for easy deployment.  It was so easy that I could deploy without really know what I was doing.  He also had a great article about using [caddy-docker](https://github.com/abiosoft/caddy-docker) to easily allow all your services to use HTTPS which I thought was great.

> Link References:
> - [Graham Gilbert Blog](https://grahamgilbert.com)
> - [Crypt](https://github.com/grahamgilbert/crypt)
> - [Crypt-Server](https://github.com/grahamgilbert/Crypt-Server)
> - [Using Caddy to HTTPS all the things](https://grahamgilbert.com/blog/2017/04/04/using-caddy-to-https-all-the-things/)

---

## What I Want To Accomplish

So, back to the issue/project at hand...  Triggered by a great talk at the [MacAdmins Conference at Penn State](https://macadmins.psu.edu) by Rick Heil, titled [Advanced Munki Infrastructure: Moving to Cloud Services](https://youtube.com/watch?v=__JXxHvuXd8), I wanted to try to streamline my munki distribution to larger clients.  In his talk, Rick teased about a Hybrid caching structure using a local nginx caching proxy server, but gave no details beyond a rough framework/idea.

We also currently use Mac-MSP Gruntwork to manage some of our clients software updates and maintenance.  Gruntwork has a built in method of doing a _staging server_ but requires a mac server onsite.  I thought maybe I could make a proxy work in place of a staging server too.

I just want a easily deployable server to place onsite which will cache and accelerate the downloads from our primary munki server.  This will reduce the cost of bandwidth and resources on the main server, allow clients to download updates faster, all while being able to do so with something as simple as a NAS.  Ideally if we could get the clients configured easily or even have them **auto-discover** the proxy that would be awesome.

---

## Components

We depend on a few open source projects to get all this done:

- [Docker](https://www.docker.com/community-edition)
- [nginx](http://nginx.org)
- [avahi-daemon](https://wiki.archlinux.org/index.php/avahi) _\*optional - for mDNS_

---

## How To Run

Most of this is ripped off of the main README of the project.  Links:
- [GitHub](https://github.com/sphen13/munki-proxy)
- [Docker Hub](https://hub.docker.com/r/sphen/munki-proxy/)

### Environment Variables

First and foremost we go over the Environment Variables.  This is how we provide settings we want to use within the container.  The only variable that is **required** is **UPSTREAM_SERVER** for obvious reasons.  The assumptions and defaults are listed in the table.

<div class="row">
    <div class="large-12 columns">
<table>
  <thead>
    <tr>
      <th>Variable</th>
      <th>Default</th>
      <th>Example</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>MUNKI_ROOT</td>
      <td> </td>
      <td>/munki</td>
      <td>Path from web root to the repo. Include first slash. Do not end in a slash.</td>
    </tr>
    <tr>
      <td>SUS_ROOT</td>
      <td> </td>
      <td>/reposado</td>
      <td>Path from web root to Apple SUS files. Include first slash. Do not end in a slash.</td>
    </tr>
    <tr>
      <td><strong>UPSTREAM_SERVER</strong></td>
      <td> </td>
      <td><code class="highlighter-rouge">https://munkiserver.example.com:8080</code></td>
      <td>Web server to be proxied including protocol. Do not end in slash. Can include port. <strong>REQUIRED</strong></td>
    </tr>
    <tr>
      <td>SERVER_NAME</td>
      <td> </td>
      <td><code class="highlighter-rouge">munkiproxy.example.com</code></td>
      <td>Set proxy web server name if needed.</td>
    </tr>
    <tr>
      <td>PORT</td>
      <td><strong>8080</strong></td>
      <td>80</td>
      <td>Port to host repo on.</td>
    </tr>
    <tr>
      <td>MAX_SIZE</td>
      <td><strong>100g</strong></td>
      <td>50g</td>
      <td>Size of munki pkgs cache. <em>The overall size may get larger than this due to how nginx functions</em></td>
    </tr>
    <tr>
      <td>EXPIRE_PKGS</td>
      <td><strong>30d</strong></td>
      <td>90d</td>
      <td>Amount of time we keep the munki <strong>pkgs</strong> directory cached for</td>
    </tr>
    <tr>
      <td>EXPIRE_ICONS</td>
      <td><strong>14d</strong></td>
      <td>7d</td>
      <td>Amount of time we keep the munki <strong>icons</strong> directory cached for</td>
    </tr>
    <tr>
      <td>EXPIRE_SUS</td>
      <td><strong>14d</strong></td>
      <td>30d</td>
      <td>Amount of time we keep the apple sus <strong>downloads</strong> directory cached for</td>
    </tr>
    <tr>
      <td>EXPIRE_OTHER</td>
      <td><strong>10m</strong></td>
      <td>1h</td>
      <td>Amount of time we keep everything else cached for <em>(catalogs etc)</em></td>
    </tr>
    <tr>
      <td>AVAHI_HOST</td>
      <td> </td>
      <td>munki-proxy</td>
      <td>mDNS hostname for proxy host.  Empty by default <strong>(mDNS disabled)</strong></td>
    </tr>
    <tr>
      <td>AVAHI_DOMAIN</td>
      <td><strong>local</strong></td>
      <td>local</td>
      <td>mDNS domain.</td>
    </tr>
    <tr>
      <td>GRUNTWORK</td>
      <td> </td>
      <td><code class="highlighter-rouge">bXVua2k6bXVua2k=</code></td>
      <td>Encoded basic auth header for upstream repo</td>
    </tr>
  </tbody>
</table>
    </div>
</div>

### Mappable Volumes

Because Docker images should be treated as immutable - we can map directories from our host system that are on persistent storage to a directory inside the container.  In this case we only have one mappable volume that we care about - and thats the cache.  We definitely want to do this because we want the cache to persist if we restart / update the container!

<div class="row">
    <div class="large-12 columns">
        <table>
  <thead>
    <tr>
      <th>Path</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>/cache</td>
      <td>Local proxy cache</td>
    </tr>
  </tbody>
</table>
    </div>
</div>

### Example Container Setup

So now that we know what we are dealing with, we can give it a shot.  The following command will start up a new container with the provided settings.  In this case we are specifying that our upstream munki server to be proxied is `https://munkiserver.example.com` and our repo is based in the `/munki` directory.  To store the cache files, we are going to use the local directory of `/var/docker/munki-proxy` to mount inside the container at `/cache`.  We are also mapping port 8080 on the host to port 8080 within the container.  `--restart-always` sets the container to auto start with the host.

``` shell
docker run -d --name munki-proxy \
	-e MUNKI_ROOT=/munki \
	-e UPSTREAM_SERVER=https://munkiserver.example.com \
	-v /var/docker/munki-proxy:/cache \
	-p 8080:8080 \
	--restart always \
	sphen/munki-proxy
```

If you wanted to pass any of the other evironment variables in you would add  `-e VARIABLE=xxxx` to the command.  As far as ports open on the host, If you wanted to use port 80 (or any other port) instead, you would change the `-p` argument to `-p 80:80` etc.

Now that you have started the container up, you can http to the host's ip or dns address at the specific port for anything under the munki repo and it will be cached and served!

### mDNS/Bonjour Discovery

For for and experimentation, I wanted to test the idea of advertising the munki-proxy container via mDNS/Bonjour.  This would potentially allow another script to automatically configure any local munki client to point to the proxy when it is available.  I have built a sample munki preflight script to auto-configure the client here:

- [munki proxy-check](https://github.com/sphen13/munki-scripts/tree/master/munki%20proxy-check) preflight script.

Please note there are potential serious security issues with this method currently.  This is just for fun/testing.
{: .notice}

When starting the container we need to specify the `AVAHI_HOST` variable as well as change the type of networking.  For this we are going to use `--net=host` instead of a single port.  This will allow the mDNS service to advertise properly to the LAN.  In the following example, we will end up with an advertised service named **munki-proxy** providing `_munki._tcp.`

``` shell
docker run -d --name munki-proxy \
	-e MUNKI_ROOT=/munki \
	-e UPSTREAM_SERVER=https://munkiserver.example.com \
	-e AVAHI_HOST=munki-proxy \
	-v /var/docker/munki-proxy:/cache \
	--net=host \
	--restart always \
	sphen/munki-proxy
```

## The Real Meat (NGINX)

The real magic of all this is done with nginx.  I learned a lot about nginx and its configuration through this exercise.  I wanted to highlight some key take-aways by walking through the configuration.

I started by doing a little googling and came across [this config](https://gist.github.com/jamesez/d61ebdde1c3a1b4e102943c21bf26acf).  Seemed like a good place to start - thanks [@jamesez](https://github.com/jamesez).  So the beginning of our config starts with:

```
worker_processes 1;

events {
  worker_connections 1024;
  use epoll;
  multi_accept on;
}

http {
  # optimize for large files
  sendfile on;
  directio 512;
  aio on;
  tcp_nopush on;
  tcp_nodelay on;

  keepalive_timeout 90;

  # open file caching
  open_file_cache            max=2000 inactive=5m;
  open_file_cache_valid      5m;
  open_file_cache_min_uses   1;
  open_file_cache_errors     on;

  # MIME type handling
  types_hash_max_size 2048;
  include /etc/nginx/mime.types;
  default_type application/octet-stream;
  types {
    application/x-plist plist;
  }

  # Don't include the nginx version number, etc
  server_tokens off;

  # Gzip Settings
  gzip on;
  gzip_min_length 1000;
  gzip_types text/html application/x-javascript text/css application/javascript text/javascript text/plain text/xml;
```

Now i added a log format to include whether we had a cache _HIT_ or _MISS_.  Might as well see how efficient we are being.

```
  # log format definition
  log_format rt_cache '$remote_addr - $upstream_cache_status [$time_local]  '
                      '"$request" $status $body_bytes_sent '
                      '"$http_referer" "$http_user_agent"';
  access_log          /var/log/nginx/access.log rt_cache;
```

Nginx can have different zones for caching.  Each `proxy_cache_path` zone can specify different settings on how large the specific cache can get, and how long files should be kept for.  I setup 4 different zones.  **MUNKI_PKGS** will grow up to 100 gigs and will keep files for 30 days.  **MUNKI_ICONS** does not have a size limit and will cache for 2 weeks.  **APPLE_SUS** will cache for 2 weeks, and **MUNKI_DEFAULT** will cache for 10 minutes.  Notice that all are using the `/cache` directory as a base - and we have passed that through from persistent storage previously.  We also specify `proxy_cache_use_stale` which tells nginx to serve up _stale_ content to the client if the upstream server is unavailable.

```
  # caching
  proxy_cache_path        /cache/pkgs keys_zone=MUNKI_PKGS:5m levels=1:2 use_temp_path=off max_size=100g inactive=30d;
  proxy_cache_path        /cache/icons keys_zone=MUNKI_ICONS:5m levels=1:2 use_temp_path=off inactive=14d;
  proxy_cache_path        /cache/apple keys_zone=APPLE_SUS:5m levels=1:2 use_temp_path=off inactive=14d;
  proxy_cache_path        /cache/other keys_zone=MUNKI_DEFAULT:5m levels=1:2 use_temp_path=off inactive=10m;
  proxy_cache_use_stale   error timeout invalid_header updating http_500 http_502 http_503 http_504;
```

We need to use `proxy_cache_valid` to set what type of HTTP return codes we want to cache and for how long.  We are going to cache return codes 202, 302 and 404 for 10 minutes by default.  `proxy_cache_revalidate` is on.  Seems like a good idea to me - not 100% on its exact behavior - but sounds like if we have a cache item that has expired, nginx can avoid downloading it again by checking if it has been changed.  `proxy_cache_lock` is an attempt at getting nginx to be a bit more efficient upstream.  If there are multiple clients requesting the same file that has not finished being cached, it will wait _(more on this later)_

```
  proxy_cache_valid         200 302 404   10m;
  proxy_cache_revalidate    on;
  proxy_cache_lock          on;
  proxy_cache_lock_timeout  5m;
```

We want to set headers upstream.  These will inherit to the `server` and `location` unless a higher level also specifies `proxy_set_header`.

```
  proxy_set_header        Host $host;
  proxy_set_header        X-Real-IP $remote_addr;
  proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header        X-Forwarded-Host $server_name;
```

Now lets define the server and set the default cache policy.  The root location is going to return a 204 which is fine.

```
  server {
    listen 8080;

    proxy_cache    MUNKI_DEFAULT;

    location = / {
      return 204;
    }
```

Finally the fun stuff... We define each `location` and set the appropriate cache policy.  Remember that higher level settings inherit (mostly) so by default each location uses **MUNKI_DEFAULT** and a 10 minute cache time.  The **catalogs** and **manifests** directory change often so lets use the default caching period.  For **icons** and **client_resources** lets cache longer and use the **MUNKI_ICONS** proxy_cache.  Once again we specify the HTTP return codes and for how long we want to cache.

```
    location /munki/catalogs/ {
      proxy_pass          https://munki.example.com/munki/catalogs/;
    }

    location /munki/icons/ {
      proxy_pass          https://munki.example.com/munki/icons/;
      proxy_cache         MUNKI_ICONS;
      proxy_cache_valid   200 302   14d;
    }

    location /munki/client_resources/ {
      proxy_pass          https://munki.example.com/munki/client_resources/;
      proxy_cache         MUNKI_ICONS;
      proxy_cache_valid   200 302   14d;
    }

    location /munki/manifests/ {
      proxy_pass          https://munki.example.com/munki/manifests/;
    }
```

Now onto the **pkgs** directory.  This one is fun.  I did a bunch of testing and found out in my testing that the _slice module_ will give us the best caching results for large files.  Without using slice, if you have a very large pkg in the repo which is not cached and you have multiple users trying to pull that at the same time, the request is actually passed upstream after a timeout waiting for the cache to fill.  This will cause unnecessary bandwidth usage.  The solution here is to treat this directory as a byte-range cache.  When nginx requests the file upstream it will do so in several requests.  In our default setup here we are slicing each pkg into 16 MB chunks.  We also need to do some modification of headers and the cache key to get this to work.

The great thing about this is, say that you have a user pulling down a macOS install pkg.  The client requests the file of the proxy normally, nginx requests the file in 16 megabyte chunks and starts filling the cache.  As the cache fills it sends the data back to the client.  If another client requests the same file a few minutes later, nginx will immediately at LAN speed fulfill the cached values so far and then continue to wait for each 16 meg slice which it then sends down to the client.  Super efficient and super cool :)

```
    location /munki/pkgs/ {
      proxy_pass          https://munki.example.com/munki/pkgs/;
      proxy_cache         MUNKI_PKGS;
      proxy_cache_valid   200 206 302   30d;
      slice               16m;
      proxy_cache_key     $host$uri$is_args$args$slice_range;

      proxy_set_header    Range $slice_range;
      proxy_set_header    Host $host;
      proxy_set_header    X-Real-IP $remote_addr;
      proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header    X-Forwarded-Host $server_name;
    }
```

Because we can I put in some rewrites and caching ability for apple SUS. This isnt really that great though and more work would need to be done if we really wanted to fully cache this info.  The reason is while we will cache the catalog files, the catalog files itself have the real upstream server or apple specified as the download location.  We cant intercept that.

```
    location / {
      if ( $http_user_agent ~ "Darwin/11" ){
        rewrite ^/index(.*)\.sucatalog$ /content/catalogs/others/index-lion-snowleopard-leopard.merged-1.sucatalog last;
      }
      if ( $http_user_agent ~ "Darwin/12" ){
        rewrite ^/index(.*)\.sucatalog$ /content/catalogs/others/index-mountainlion-lion-snowleopard-leopard.merged-1.sucatalog last;
      }
      if ( $http_user_agent ~ "Darwin/13" ){
        rewrite ^/index(.*)\.sucatalog$ /content/catalogs/others/index-10.9-mountainlion-lion-snowleopard-leopard.merged-1.sucatalog last;
      }
      if ( $http_user_agent ~ "Darwin/14" ){
        rewrite ^/index(.*)\.sucatalog$ /content/catalogs/others/index-10.10-10.9-mountainlion-lion-snowleopard-leopard.merged-1.sucatalog last;
      }
      if ( $http_user_agent ~ "Darwin/15" ){
        rewrite ^/index(.*)\.sucatalog$ /content/catalogs/others/index-10.11-10.10-10.9-mountainlion-lion-snowleopard-leopard.merged-1.sucatalog last;
      }
      if ( $http_user_agent ~ "Darwin/16" ){
        rewrite ^/index(.*)\.sucatalog$ /content/catalogs/others/index-10.12-10.11-10.10-10.9-mountainlion-lion-snowleopard-leopard.merged-1.sucatalog last;
      }
      if ( $http_user_agent ~ "Darwin/17" ){
        rewrite ^/index(.*)\.sucatalog$ /content/catalogs/others/index-10.13-10.12-10.11-10.10-10.9-mountainlion-lion-snowleopard-leopard.merged-1.sucatalog last;
      }
    }

    location /content/catalogs/ {
      proxy_pass          https://munki.example.com/content/catalogs/;
    }

    location /content/downloads/ {
      proxy_pass          https://munki.example.com/content/downloads/;
      proxy_cache         APPLE_SUS;
      proxy_cache_valid   200 206 302   14d;

      slice               16m;
      proxy_cache_key     $host$uri$is_args$args$slice_range;
      proxy_set_header    Range $slice_range;
      proxy_set_header    Host $host;
      proxy_set_header    X-Real-IP $remote_addr;
      proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header    X-Forwarded-Host $server_name;
      proxy_http_version  1.1;
    }
  }
}
```

## Whats Next

Well, next would be to make sure this works really well beyond my limited testing.  Beyond the basics the bigger tasks I see that need work:

- Tackle security with the mDNS/autoconfigure option.
  - Perhaps the client and server could test for a specific validation before switching the munki config.  Maybe this could be solved by a client/server certificate that could be installed on the client and passed to the docker container.  Would love any ideas :)
- Apple SUS caching is implemented in this because it was simple to add.  It is rather ineffective though since the catalog itself will reference the real upstream server or apple itself which will bypass the proxy.  Would be nice to capture this cache too.
  - Idea would be maybe to pull the catalog(s) down periodically and use something like `sed` to modify the server url to point to the proxy
