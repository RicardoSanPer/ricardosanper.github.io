---
layout: single
title: Home Server - Network Hardening
categories: writeups homelab
toc: enabled
toc_sticky: enabled
---

It's been almost half a year since I set up a small home server and started self-hosting services. I began by hosting Jellyfin and other media management services as my library was disorganized and it was a hassle to move files between devices when I wanted to watch something. Nowadays I host over a dozen services that I find useful.

However, I am still using the server's IP address and port to access the services, which has been bothering me for several reasons:
- Some services need to talk to each other. For this they use the server's IP, which means traffic is unnecesarily routed through the LAN when, in theory, it shouldn't need to leave the localhost.
- It is inconvenient to manage ports when hosting a new service, not to mention a security risk to have so many ports exposed.
- The services use HTTP, and I'd like to use HTTPS. I'd also like to use proper domain names to access the services.

My home server is not exposed to the public internet, but the other reason I set one up was to learn about server administration and security, so I want to do something about it.

# Reverse Proxies
The solution to my problem is a reverse proxy. A proxy is a program that acts as an intermediary between a client and a server. A proxy can either be forward, for requests that leave the client, or reverse, for requests that arrive at the server.

A reverse proxy can solve my issues, as it can receive incoming requests and forward it to the corresponding service. This is like having a waiter take your order instead of directly asking the chef for food and the bartender for drinks. And since the proxy acts as a single access point for all the services in the backend, then I only need to set up HTTPS for the proxy and not for every single service. 

And finally, the reverse proxy maps domains and subdomains to the services.

# Docker Networking
Before actually setting up a reverse proxy I first decided to clean up the Docker networking. My current setup uses the default networking configuration, which according to the Docker documentation, uses the default ```bridge``` network driver. Other network drivers include ```host```, which allows containers to use the host's network interface directly, and ```none```, which entirely removes all networking from the container. ```bridge``` is the recommended one as it adds a degree of isolation between a container and the host.

```bridge``` also has some differences whether you use the default bridge configuration or a custom defined one. The main ones for my case are:
- Containers on the default bridge can only communicate with each other by IP addresses. User-defined bridges can communicate by using the container name, as they provide automatic DNS resolution. This would solve my concern about unnecesary router traffic.
- User defined bridges provide better isolation, as they essentially work as their own little subnet. 

This is quite useful for containers that must communicate with each other, but there's still one problem: Some containers must communicate with host services an viceversa. For this, containers can use ```host.docker.internal``` to communicate from containers to the host, adding it as an extra host on the docker compose file. 

For starters I began with creating separate networks for different stacks of services using ```docker network create NAME```. I created the following networks:
- **arrnet** for my media stack, which includes the \*arr stack, shoko server
- **tools** for various tools like Cyberchef, Omnitools, and dashboards
- **productivity** for productivity services like Kanboard and FreshRSS
- **gitea** for Gitea and Gitea runners

Then I simply changed the docker compose files to use the networks by adding:
~~~
networks:
    - NETWORK_NAME
~~~
to the Docker compose files and checking that the services continued to work. To test that the containers could make use of the networks I went to the configuration of one, swapped an URL (```http://<OTHER_CONTAINER_IP>:port```) it uses to connect to another container for its name (```http://OTHER_CONTAINER_NAME:port```), and tested the connection. It worked without issues, so I went around applying the changes to other services.

The next step was to connect containers with outside services. Since it is only a handful that need this type of connectivity I set to only modify those. I simply had to add:
~~~
extra_hosts:
    - "host.docker.internal:host-gateway"
~~~
to the docker compose files of the containers. Swapping ```host.docker.internal``` in place of the host's IP worked, so I went around applying the new configuration.

Next was connecting host services to Docker containers. This was as simple as swapping IPs with ```localhost/127.0.0.1``` and the port. Thankfully only one service required this change in configuation.

Finally, I deleted the default networks Docker created for the images.

# nginx
With the Docker networking cleaned up I installed nginx. To create set up the reverse proxy I created a configuration file in ```/etc/nginx/conf.d```. This configuration will contain all reverse proxy mappings with the base configuration:

~~~
server
{
        listen PORT;
        location / {
                proxy_pass http://127.0.0.1:PORT;

                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }
}
~~~
The changes can be applied with ```nginx -s reload```.

This currently seemed to work, but the purpose is to reduce the exposed ports and this configuration is not so different from what I already had before, so the next step was to set up a DNS server to resolve a local domain, something else I wanted to try.

# Adguard
I decided to install Adguard as my DNS solution. I installed it using a Docker container, in it's own ```adguard``` network. However, upon trying to run it I got an error about the port 53 being already in use.

Using ```sudo lsof -i :53``` I can check the services using it, being ```systemd-resolve```. The port can be freed by editing ```/etc/systemd/resolved.conf``` and uncommenting ```DNSStubListener```, changing the value to ```no```. Restarting the service with ```sudo systemctl restart systemd-resolved``` results in no output when executing again ```sudo lsof -i :53```, meaning the port is free.

After finishing the above configuration I accessed the Adguard web interface and finished the setup. To test I set my personal computer to use the homeserver as DNS server, and was pleasantly surprised to see it was generting activity on the admin panel, meaning it was working properly.

Now that nginx and Adguard were working, it was time to start creating custom URLs. For that I created custom filter in Adguard's admin panel, mapping domains and subdomains in the format ```service.homelab.rsp``` to my server's IP. Then I simply added server blocks in the nginx configuration file to pass the requests to the proper services and configured the services to use the new URL where needed.

# More Hardening 
After testing that the domain resolution worked and that the services continued to work as intended I set to update the services to only listen on localhost. For the Docker containers Docker I simply had to add ```127.0.0.1``` to the port mappings. For services on the host I modified the configuration according to their respective documentations, but generally it consisted of swapping some the application IP from ```*``` or ````0.0.0.0``` to the local host.

Further application configurations I had to change for some services was adding the new URL so that they could properly load content on the browser, as they were no longer using a raw IP address.

Lastly all that was left was to update the firewall rules with UFW. I started by revoking the previous rules that allowed traffic in all the ports of the previously exposed services, then fine tunning to allow the traffic only between the services that reqired it, like between Shoko server and Jellyfin, or the *arr stack and qbitTorrent.

-- TODO: finish network diagram
-- Firewall rules
-- Mention per tool configs?
# References
Docker Networking Overview: [Networking Overview](https://docs.docker.com/engine/network/)
Nginx Installation Guide (Linux): [nginx: Linux Packages](https://nginx.org/en/linux_packages.html)
Nginx Beginners's Guide: [Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)