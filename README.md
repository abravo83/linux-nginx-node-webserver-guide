# INSTRUCTIONS FOR SETTING UP A LINUX SERVER AS A NODE WEB SERVER WITH MULTIPLE EXPRESS SERVERS

## The Problem

We want to have an Express web server or similar running through a node process.

The problem is that a web server only responds to port 80 for http or port 443 for https by default.

If we configure an Express server that, for example, runs a web application made with Angular or React and then we want to add another web application that has an API and also works listening to port 443 and 80. When we make the request to the server... Which of the express-node servers responds?

![Figure 1: A server with multiple nodes listening to port 443](img/fig-1.png)

What if we add even more?

The problem will therefore be that our server won't know which node to respond from. If we've added a domain `www.mywebdomain.com` and even if we add a second domain `www.myapidomain.com`, it won't matter, since domains only indicate which IP to direct requests to, and not which port. Therefore, we cannot expect each domain to be answered by different Express servers.

## The Solutions

- One solution would be to get a different server per web application. But this will be a costly and inefficient solution.

- Another solution could be to specify different ports and have the web servers not listening to the same port. This solution is perhaps more feasible for an API, leaving the React/Angular web on 443 so it can be called from `https://www.mywebdomain.com` (When no port is specified, it goes to 443 if it has https and 80 if it has http) while API requests could be specifically directed to other ports: `https://www.mywebdomain.com:3030`. This solution is valid in principle, but if we add more websites serving frontends, we cannot expect users of those websites to use domains that specify ports. It's not a professional solution. Additionally, this solution will bring us other problems when issuing and renewing SSL certificates to use https://

- The solution we're going to propose here is to use a type of web server that is the only one listening to those ports 443 and 80 and forwards those requests to other web servers running locally on the server on different ports (e.g., 3010, 3011...) that won't be directly exposed. This type of Web Server is known as a **Reverse Proxy**. If we also combine it with the use of subdomains, we achieve with a single domain the ability to make many web services available: `https://www.mywebdomain.com`, `https://api.mywebdomain.com`, `https://mywebapp2.mywebdomain.com` since the reverse proxy will analyze which domain each request comes from and distribute it to the appropriate local web server. It's also a common solution that will work correctly with ssl certificate management clients like _certbot_ and will allow us to automate the certificate renewal process.

![Figure 2: A server listening with a reverse proxy on port 443](img/fig-2.png)

## The Configuration Process

The process of configuring a web server from scratch goes beyond setting up a reverse proxy and multiple node servers running locally. But not much beyond. We'll cover the complete process since we all encounter the complete process and it's helpful to have the steps in a single guide.

This process aims to configure a server that is secure and functional for production, so some steps may seem unnecessary, but they are done to improve security.

You might wonder what web server to have. My recommendation is Ubuntu Server or Rocky Linux in its latest LTS version and WITHOUT desktop environment to avoid wasting resources unnecessarily since we'll perform operations through the terminal.

Before acquiring or contracting the server, whether it's a VPS or a cloud server instance, it's HIGHLY recommended that you're prepared to follow at least the first steps prior to installing web services since once the server is up, you can be sure that some automated bot will find your web and try to login by brute force using default users (like root) and common passwords.

That's why our first steps are aimed at this:

1. Find and analyze the access data we've been given to access our web server.
2. Modify access data and the possibility of access from known accounts (e.g., 'root', 'admin', 'administrator'...)
3. Check and Set up the firewall with necessary exceptions so we can continue connecting to the server.
4. Update --Up to here are the urgent operations to perform right after starting the server--
5. Install Node
6. Install NGINX (Our Reverse Proxy)
7. Install MySQL or MongoDB or whatever DB server we're going to use (If we need them)
8. Create a user WITHOUT SUDO PERMISSIONS who will be the owner of the folders where our node servers are contained.
9. Create BASH scripts with commands to start the node servers.
10. Create services that execute these scripts so our node servers function as server services.
11. Configure our domains and subdomains to direct to our server.
12. Configure our reverse proxy to direct requests from each domain or subdomain to their corresponding local node servers.
13. Configure certbot to issue and remain in charge of renewing SSL certificates when appropriate.
14. Check everything (Services enabled, Server restart with everything automatically up) and go touch some grass.
