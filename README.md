## Introduction
This repository contains examples of the nginx configuration files that I used to set up my reverse proxy. This proxy hides everything behind it, the Pterodactyl panel, the node servers and the game services. It should be noted that I'm no expert, I can't guarantee this will work for you as I have not tested them thoroughly.

**You should make sure you backup your nginx before modifying it** `zip -r /etc/nginx.zip /etc/nginx`

## My server design
You should probably be aware of how my servers are configured for this to make the most sense to you. Each service is its own virtual machine running ubuntu server.
- Opnsense Router           -       IP 172.16.0.1
- Nginx Proxy Server        -       IP 172.16.0.100
- Pterodactyl Panel Server  -       IP 172.16.0.101
- Pterodactyl Node Server    -       IP 172.16.0.102

The main thing you should know is the opnsense router is only port-forwarding to the nginx proxy server, there is no direct line between the outside world and the two Pterodactyl servers.

## Requirements
- Nginx server running Nginx version 1.25 or greater. - [Nginx update guide](https://developerinsider.co/install-update-nginx-to-the-latest-stable-version-on-ubuntu/)
- Pterodactyl Panel server
- Pterodactyl Node server
- Understanding of port forwarding and how that will affect your security

### Notes regarding Nginx
There are a dozen different ways to configure nginx, the way I prefer and use is using `sites-available` and `sites-enabled` directories.

## Pterodactyl Panel configuration
In order for the panel to work with a proxy you will need to modify the `.env` that is located in the root directory of the panel <sub>(default location `/var/www/pterodactyl/.env`)</sub>
You will need to add the following line to the `.env` file making sure to replace {IP} with the ip address of the proxy i.e `172.16.0.100` alternatively you can use a wildcard and replace {IP} with `*`
```
TRUSTED_PROXIES={IP}
```
**Make sure you can connect to your panel via HTTPS before continuing.**

## Pterodactyl Node configuration
Head over to `YourPanel/admin/nodes` and either create a new node or select a node to modify, for the most part you will want to follow the [guide on the Pterodactyl website](https://pterodactyl.io/wings/1.0/installing.html#configure) the main thing you want to make sure is that you have `Behind Proxy` set to... well... `Behind Proxy`

## Nginx.conf
First of all, you will want to create two new directories `/etc/nginx/streams-available` and `/etc/nginx/streams-enabled` as these will be used later.

Now take a look at the [nginx configuration file](nginx.conf). The most important bit to take note of  is the `stream` block as this is what handles the UDP forwarding. The bare minimum required for it to work is:
```
stream {
    include /etc/nginx/streams-available/*.conf
}
```
What does this do? Basically this is telling nginx to include any file in `/etc/nginx/streams-enabled` that end with the `.conf` extension into the stream block.

Inside the `http` block you should have an `include` for simplicity sake, it should be `include /etc/nginx/sites-enabled/*.conf` if it's not you will need to keep that in mind when updating my configuration files.

## streams-available
Take a look at [proxy.domain.conf](streams-available/proxy.domain.conf) it's a tad confusing on the surface, but pretty much what it's doing is listening on the port range `6700-6720` <sub>(why these ports? because they are the ports I've assigned to my node, change them to suit your needs)</sub> using `udp` protocol and to `reuseport` <sub>([what is reuseport?](http://nginx.org/en/docs/http/ngx_http_core_module.html))</sub> much like with a standard nginx config you are then providing it a `server_name` I like to use both a FQDN and the public IP of the proxy server, here's the bit you need to remember, the proxy_buffer may need tuning to your specific needs <sub>([find out more about proxy_buffer_size](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffer_size))</sub>. I set my buffer to 128k as that is what worked best for me.

**Don't forget to symlink your config!** `ln -s /etc/nginx/streams-available/proxy.domain.conf /etc/nginx/streams-enabled`

## sites-available
First site config is [redirect.ssl.conf](sites-available/proxy.domain.conf), its a basic http to https redirect. That's it.

Next up is [panel.domain.conf](sites-available/proxy.domain.conf), another basic one, this is just the configuration file for the Pterodactyl panel, it can also be found [here on the Pterodactyl website](https://pterodactyl.io/panel/1.0/webserver_configuration.html#nginx-without-ssl)

Finally it's the hard one [proxy.domain.conf](sites-available/proxy.domain.conf). The first server block listens for any connection on ports 443, 8080 and 2022 (aka https, node api, sftp ports) and provides them with an SSL certificate <sub>I used certbot</sub>. The second server block is a bit more specific, it is listening to any connection on the https port for that server_name, for example `https://panel.yourdomain.com` if has a match, it will proxy it off to whatever address you provide it, for my setup I set that to `http://172.16.0.101` its important to note that the `http://` or is required for it to function. The third block pretty much does the same thing, it listens to ports 8080 & 2022 and forwards them to the node server, again you will need to provide it a `proxy_pass` address, as an example in mine it was set to `http://172.16.0.102`

**Don't forget to symlink your config!**
```
ln -s /etc/nginx/sites-available/proxy.domain.conf /etc/nginx/sites-enabled`
ln -s /etc/nginx/sites-available/redirect.ssl.conf /etc/nginx/sites-enabled`
```

