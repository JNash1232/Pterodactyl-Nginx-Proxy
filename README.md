## Introduction
This repository contains examples of the nginx configuration files that i used to set up my reverse proxy. This proxy hides everything behind it, the pterodactyl panel, the node servers and the game services them self. It should be noted that I'm no expert, I can't garentee this will work for you as I have not tested them thoroughly.

## My server design
You should probably be aware of how my servers are configured for this to make the most sence to you. To start each service has its own virtual machine.
- Opnsense Router           -       IP 172.16.0.1
- Nginx Proxy Server        -       IP 172.16.0.100
- Pterodactyl Panel Server  -       IP 172.16.0.101
- Pterodactyl Node Sever    -       IP 172.16.0.102

The main thing you should know is the opnsense router is only portforwarding to the nginx proxy server, there is no direct line between the outside world and the two pterodactyl servers.

## Requirements
- Nginx server runing Nginx version 1.25 or greater. - [Nginx update guide](https://developerinsider.co/install-update-nginx-to-the-latest-stable-version-on-ubuntu/)
- Pterodactyl Panel server
- Pterodactyl Node server
- Understanding of portforwarding and how that will effect your security

### Notes regarding Nginx
There is a dozen different ways to configure nginx, the way i prefer and use is using `sites-available` and `sites-enabled` directories.


## Nginx.conf
First of all you will want to create two new directories `/etc/nginx/streams-available` and `/etc/nginx/streams-enabled` as these will be used later.

Now take a look at the [nginx configuration file](nginx.conf). The most important bit to take note of  is the `stream` block as this is what handles the UDP forwarding. The bare minimum required for it to work is:
```
stream {
    include /etc/nginx/streams-avilable/*.conf
}
```
What does this do? Basically this is telling nginx to include any file in `/etc/nginx/streams-enabled` that end with the `.conf` extention into the stream block.

Inside the `http` block you should have an `include` for simplicity sake it should be `include /etc/nginx/sites-enabled/*.conf` if its not you will need to keep that in mind when updating my configuration files.

## streams-available
Take a look at [proxy.domain.conf](streams-available/proxy.domain.conf) its a tad confusing on the surface, but pretty much what its doing is listening on the port range `6700-6720` <sub>(why these ports? becuase they are the ports i've asigned to my node, change them to suite your needs)</sub> using `udp` protocol and to `reuseport` <sub>([what is reuseport?](http://nginx.org/en/docs/http/ngx_http_core_module.html))</sub> much like with a standard nginx config you are then providing it a `server_name` i like to use both a fqdn and the public IP of the proxy server, heres the bit you need to remember, the proxy_buffer is the most important bit to this and may need tuning to your specific needs <sub>([found out more about proxy_buffer_size](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffer_size))</sub>. I set my buffer to 128k as that is what worked best for me.

**Don't forget to symlink your config!** `ln -s /etc/nginx/streams-available/proxy.domain.conf /etc/nginx/streams-enabled`

## sites-available
Frst site config is [redirect.ssl.conf](sites-available/proxy.domain.conf), its a basic http to https redirect. Thats it.

Next up is [panel.domain.conf](sites-available/proxy.domain.conf), another basic one, this is just the configuration file for the pterodactyl panel, it can also be found [here on the pterodactyl website](https://pterodactyl.io/panel/1.0/webserver_configuration.html#nginx-without-ssl)

Finally its the hard one [proxy.domain.conf](sites-available/proxy.domain.conf). The first server block listens for any connection on ports 443, 8080 and 2022 (aka https, node api, sftp ports) and provides them with an SSL certificate <sub>I used certbot</sub>. The second server block is a bit more specific, it is listening to any connection on the https port for that server_name, for example `https://panel.yourdomain.com` if has a match, it will proxy it off to whatever address you provide it, for my setup i set that to `http://172.16.0.101` its important to note that the `http://` or is required for it to function. The third block pretty much does the same thing, it listens to ports 8080 & 2022 and forwards them to the node server, again you will need to provide it a `proxy_pass` address, as an example in mine it was set to `http://172.16.0.102`

**Don't forget to symlink your config!**
```
ln -s /etc/nginx/sites-available/proxy.domain.conf /etc/nginx/sites-enabled`
ln -s /etc/nginx/sites-available/redirect.ssl.conf /etc/nginx/sites-enabled`
```

