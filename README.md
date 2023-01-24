# ---GUIDE IS A WORK IN PROGRESS---

# Purpose
 A guide on how to get SSL certificates working on an Unraid service using a custom domain and Tailscale.

 I'm almost positive this isn't the best way to do this, but it's the way I managed to make it work after 2 days and 15 hours of troubleshooting. This guide is mainly meant to start a discussion. If you see ways to improve this, shoot me a pull request. Frankly speaking the only documentation I could pull up was a couple reddit posts from months/years ago that never got the problem solved.

# Quick Notes

## Disclaimer
I'm not a network engineer. I'm not a cloud architect. I'm a dude with an Unraid box and far too much free time. The advice here is by no means given with any kind of warrenty or guarentee, implied or otherwise. If you break something, or expose something, or leak something you shouldn't, that's on you. Anything below this point should be considered, "Just my opinion, man." and used in consideration

## "Why bother with Tailscale if you're going this far?"
As I mentioned above, I'm just a dude who's stumbled into the deep end of self-hosting and got a bit carried away. I don't know what I don't know. I keep all my services behind Tailscale so I don't have such a large attack surface. I also don't always have time to keep things 100% up to date, so keeping stuff behind TS can help smooth over times where I might fall behind updating things like Nextcloud. Also, DNS and routing is **hard** and trying to deal with double NAT issues and the like just isn't something I want to do.

That being said, for most things this is probably overkill. Tailscale already encrypts your traffic. The main reasons I ended up doing all this was to remove the annoying "Insecure Connection" messages, stop having to tell my family to ignore the certificate errors (something I really don't want them doing, generally), and allow my girlfriend to sync her Joplin notes to our server from her iOS device.

# Assumptions
- You are running Unraid v6.11.5 or greater (Guide written on v6.11.5)
- You already own a custom domain (example.com will be used in the guide)
- All services will be running on the same Unraid box
- You have the Docker ("Community Apps") plugin installed and working on Unraid
- You know what a "Docker Container" is and how it works
- Tailscale is already up and running on Unraid and you can remotely access everything via the web
- Services you want to access (Nextcloud, Paperless-ngx, Joplin Server) are already up and running
- You know very basics of IP/Networking and are vaugly familiar with terms like "DNS" "Proxy" "VPN" and "SSL"
- You have API access to your DNS provider

# Overview
1. Configure Unraid so it doesn't get in the way while we're trying to setup the reverse proxy.
2. Install and configure Nginx Proxy Manager (NPM) to handle routing all of our traffic to the correct service.
3. Configure our DNS record so traffic is sent to the Tailscale IP address of our server.
4. Use DNS challenges in NPM to get Letsencrypt certificates for our services to make the annoying "Insecure Connection" messages finally go away.

# Step 1: Move Unraid GUI interface to New Port
Before we get too far along, we'll need ports 80 & 443 (HTTP and HTTPS ports) open for our reverse proxy. By default, Unraid uses these ports for the GUI to allow for remote monitoring/configuration without having to SSH into it all the time. 

Most guides won't mention this step as they generally assume you're exposing ports on your router and your router can foreward ports 80 & 443 traffic to whatever port Nginx Proxy Manager (NPM) is using. We're using Tailscale and by default all traffic from our client devices will come directly to the server on ports 80 & 443, so we have to change this and put NPM here instead.

1. Login to your Unraid GUI by entering the IP address of your server.
2. Navigate to `Settings > Managment Access`
3. Make sure `Use SSH` is **Enabled** and the port is set to `22`
    - This will be our backup into the server should something go wrong and we can't access the GUI. This is usually on by default.
4. Change `HTTP port:` from `80` to something else like `90` or `10080`
    - **Write down what you change this to!** If you forget this number, you won't be able to access the Unraid web GUI.
    - Make sure this port isn't conflicting with ports used by your Docker containers.
5. Change `HTTPS port:` from `443` to something else like `453` or `10443`
    - Write this one down too!
    - Make sure this port isn't conflicting with ports used by your Docker containers.
6. You should be disconnected from the Unraid web GUI after a moment. Login to the web GUI using the **new** port by typing `X.X.X.X:[Port Number]`
    - `X.X.X.X` = The IP address of your Unraid server
    - `[Port Number]` = The port number you assigned to HTTP in step 4.

Congradulations! The Unraid GUI is now out of the way and we can get Nginx Proxy Manager up and running to handle the incoming traffic from Tailscale.

## TODO:
- Add section detailing SSH recovery if this hangs up or goes wrong for whatever reason.

# Step 2: Nginx Proxy Manager Up and Running
Next we'll be using Nginx Proxy Manager (**NPM** for short) to handle all the incoming traffic and route it to where it needs to go. [The benefits of reverse proxy servers are pretty well documented.](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/) I won't bore you with the details, but the main reason we're using it is to centralize the routing of our traffic. NPM also does a great job of simplifying the SSL and HTTPS part of this whole setup.

1. In the Unraid GUI, go to `Apps` and search for `Nginx Proxy Manager`
2. Find the one named "NginxProxyManager" from "Djoss's Repository"
    - [Here's the link to the Docker container on GitHub.](https://github.com/jlesage/docker-nginx-proxy-manager)
    - That GitHub should be the source of truth on documentation and information about this specific container. If you are curious about what certian variables do and want to tweak things, this is an great place to start.
    - DJoss is simply the current **maintainer** of this docker container. If you are having trouble finding the container, this may have changed. Use information from the GitHub and search results to determine the new maintainer. Then maybe let me know so I can update this :)
3. Click the app, then click `Install`
4. Set `HTTP Port:` to `80` and `HTTPs Port:` to `443`
    - Double check that the `Web UI Port:` isn't conflicting with any of your other Docker containers. You can change it if you like, just remember what it is so you can access is later.
    - If you are using custom Docker networks, set the `Network Type:` variable accordingly.
5. Click `Done` and wait for Docker to pull everything down and start up the container.
6. 

# Step 3: Custom Domain Resolution
Even before setting up NPM, we didn't **need** custom domain names. Each of our services is accessable by typing in the Tailscale IP address of the server then appending it with `:[Port Number]`. However, this looks awful in practice and can make it harder for less tech-savvy users to get things setup. To do this, we're going to point our domain name to the Tailscale IP of our server.

"Can you really just point a DNS entry to a Tailscale IP?" You might ask.

Yes, yes you can.

Don't take my word for it. [There's a whole page about DNS and using it with Tailscale.](https://tailscale.com/kb/1054/dns/)

**NOTE:** *These steps are intentionally vauge. Every DNS Manager is going to be different and you'll have to interperet the steps and instructions here for your use case. Generally speaking, we're adding an A record to our DNS record and pointing that to the Tailscale IP of our Unraid server.*

1. Find the Tailscale IP address of your Unraid server
    - Easiest way to do this is by going to https://tailscale.com and finding your Unraid server in the list of "Machines" in the dashboard. 
2. Login to your DNS manager.
    - This is usually the place you got your domain name from (GoDaddy, NameCheap, Google Domains, etc).
3. Find the page that let's you to edit records for the domain you want to use.
4. Create a new `A` record
    - `Hostname` = `@`
    - `IP Address` = The Tailscale IPV4 address of the Unraid server
    - `TTL` = `300`? (It depends)
        - [Here's some documentation on how to set TTL.](https://www.varonis.com/blog/dns-ttl) Frankly speaking, I don't know enough about DNS to be able to give a solid reccommendation for setting a TTL. I have mine set to 24 hours and that's never been a problem, but YMMV.
5. Allow the DNS record update to propigate to other DNS servers
    - This could take up to 48 hours.
    - In my experience, Cloudflair DNS servers (`1.1.1.1`) tend to pick up changes rather quick (~5 minutes). Changing you DNS server to them might reduce wait time.
6. Test the routing by entering your domain (`example.com`) into a Tailscale connected device.
    - You should see the landing page for NPM.
    - If you disconnect from Tailscale and reload the page, the NPM landing page should no longer load.
7. For each service you wish to host, create a new `A` record
    - `Hostname` = Whatever you want for a particular service.
        - Example: `nextcloud` or `cloud` for a Nextcloud instance
        - Example: `dashboard` or `dash` for a dashboard like Heimdall
        - Example: `joplin` for a Joplin server
    - `IP Address` = The Tailscale IPV4 address of the Unraid server
    - `TTL` = 300? (See above)
8. Open the NPM GUI
9. Navigate to `Hosts > Proxy Hosts`
10. Click `Add Proxy Host` in the top right
11. Enter this information into the dialog box
    - `Domain Names` = the full domain name you want your service to have
        - Make sure to hit Tab or Enter after you finish typing this. Not doing so will make the entered text disappear and cause and error.
    - `Scheme` = weather this service accepts `http` or `https` traffic on their port
    - `Foreward Hostname/IP` = The Tailscale IP address of the service (usually your Unraid server)
    - `Foreward Port` = The port the service is hosted on
        - Can be found by looking at your Docker containers in the Unraid GUI
    - `Cache Assets` = Weather you want NPM to cache assets for the service
    - `Block Common Exploits` = `On` (Sometimes this can break things but I turn it on by default)
    - `Websockets Support` = Turn on if your service needs Websockets support
    - `Access List` = `Publically Accessable`

You should now be able to access your services via the custom domains you entered.


# Step 4: SSL and HTTPS
Full disclosure, this last step is the part I'm the least familiar with. As I understand it, the standard way to setup Lets Encrypt SSL certificates involves the remote Let's Encrypt server validating your ownership by finding a file that is accessable on your server. Obviously, this won't work if we're behind Tailscale. Instead, we're going to use a [DNS challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge) to authenticate our ownership of the server.

Instead of putting a file on our server, we're going to make a DNS record change and add a string of text given to us by Let's Encrypt. Since Let's Encrypt didn't see the string of text on there before asking us for it, and it's there afterwards, it's a safe assumption that we added it there ourselves. And if we added the text record, then we must own the domain.

Now, most guides I managed to dig up on this, mentioned using a bit of software called certbot. [Many,](https://www.geeksforgeeks.org/using-certbot-manually-for-ssl-certificates/) [many,](https://dev.to/michaeldscherr/lets-encrypt-ssl-certificate-via-dns-challenge-8dd) [many,](https://cloudness.net/certbot-dns-challenge/) [many,](https://www.google.com/search?q=dns+challenge+letsencrypt) guides suggest the use of certbot. So much so that it feel like a bit of an industry standard to use.

I didn't use certbot.

I used the SSL certificate managment system in NPM. Based on the error reports I read after messing this up several times, NPM just uses certbot under the hood, and since we already have NPM setup, we'll just use that.

1. Open NPM and navigate to the `SSL Certificates` section
2. Click `Add SSL Certificate`
3. Enter the information for each field
    - `Domain Names` = The full domain name, including subdomains, used for a service
        - Example: cloud.example.com
    - `Email Address for Let's Encrypt` = The email address for Let's Encrypt to contact you
        - This needs to be a **real** email address. Let's Encrypt will send you an email if there's something wrong so make sure it's being monitored.
    - `Use a DNS Challenge` = `On`
    - `DNS Provider` = Your DNS Provider
    - `Credentials File Content` = Similar to making DNS changes or forewarding a port on your router, there are alot of variables here. Consult your DNS Provider's documentation and the information provided in the text box.
    - `Propogation Seconds` = Can be left blank
4. Accept the Terms and Conditions of Let's Encrypt
5. Click `Save`
6. Wait for the system to process your request
    - This might take a couple minutes so be patient
7. Your certificate should be visable on the screen now
8. Navigate to `Hosts > Proxy Hosts`
9. Select your domain for that certificate and click the three dot menu on the right, then `Edit`
10. Click `SSL` at the top of the dialog box
11. Select your SSL certificate for the associated domain from the dropdown menu
12. Repeate steps 1-11 for each subdomain you're using


# The Problem with Tailscale's MagicDNS and HTTPS Solutions
Tailscale (the company) is quite aware of the issue and difficulties with setting up HTTPS on connected servers. To address this, they've 

# 