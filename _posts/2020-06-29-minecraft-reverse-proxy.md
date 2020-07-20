---
layout: post
title: Set up Reverse Proxy for Minecraft Servers
tags: [how to]
---

This post is about how to set up a reverse proxy for Minecraft server (or for TCP in general) with Nginx.

I have done this on Ubuntu 18.04 with Nginx and `libngxin-mod-stream` installed from the default repositories using `apt`. I set up the reverse proxy for some private Minecraft servers running Spigot 1.15.2 and Paper 1.15.2 with Waterfall, and it does not require any changes on the Minecraft server. **Many of the public servers have rules that DO NOT allow connecting to their servers through a proxy**.

## Why?

![Reverse proxy can improve connection.](/assets/image/minecraft-reverse-proxy.png)

When playing with friends on the Minecraft server `server.example.com`, if the route between someone's client and the server has a bad connection with high ping and high packet loss, the gaming experience would not be great or even not able to connect.

However, if there is be a server location with better connections to both that client and the server, and it could possibly be used as a proxy to improve the gaming experience.

## How?

### Prerequisite

Nginx and `libnginx-mod-stream` installed (either from package repositories or compiled from source).

### Configurations

Assuming the Minecraft server is running on `server.example.com` and port `25565`.

Some of the commands might require `sudo` privilege.

On the sever used for reverse proxy, edit the Nginx config file, normally located at `/etc/nginx/nginx.conf` on Ubuntu if installed with `apt`, add the config below:

```
stream {
    server {
        # Port number the reverse proxy is listening on
        listen  25565;
        # The original Minecraft server address
        proxy_pass  server.example.com:25565;
    }
}

```

Reload Nginx:

```
service nginx reload
```

In case firewall is enabled, run the following command to allow connections:

```
ufw allow 25565/tcp
```

### Connect to the reverse proxy

To connect via the reverse proxy, connect with Minecraft client as usual, however use the proxy server's address and port number instead of the original server's, and in this case it is `proxy.example.com:25565`. Clients are still able to connect directly to the original server without using the reverse proxy with the original address and port number.

If the port number is the default 25565, it can be omitted in the client. In case the port number is not the default 25565, it is possible to [set up a SRV record on DNS](https://www.spigotmc.org/threads/guide-setting-up-srv-records.52303/) to avoid specifying the port number in the client (this step is optional).

## Conclusions

There are several advantages setting up the reverse proxy this way:

- It does not require to modify anything on the Minecraft server itself, as long as it allows the clients to connect to it through a proxy.
- It does not require any special setup on the clients, just to use the reverse proxy's address instead of the original Minecraft server's address when connecting in Minecraft.
- It works for many different Minecraft server and client version.
- It still allows the clients to connect directly to the Minecraft server. It is useful when some of the clients has better connection to the original server.

This setup is easy to understand and config, also requires little server resource. It is also possible to use access control from Nginx or iptables etc..

There are limitations of this setup too. It simply works as a TCP reverse proxy, and it does not do some fancy stuff like connecting multiple Minecraft servers together. Also again, **it SHOULD NOT be used if the Minecraft server has rule that does not allow clients to connect through a proxy**.
