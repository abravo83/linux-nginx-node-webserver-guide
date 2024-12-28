# INSTRUCTIONS FOR SETTING UP A LINUX SERVER AS A NODE WEB SERVER WITH MULTIPLE EXPRESS SERVERS

## The Problem

We want to have an Express or similar web server running through a node process.

The problem is that a web server only responds to port 80 for http or port 443 for https by default.

If we configure an Express server that, for example, runs a web application made with Angular or React, and then we want to add another web application that has an API and also works by listening to port 443 and 80. When we make the request to the server... Which of the express-node servers responds?

![Figure 1: A server with multiple nodes listening to port 443](img/fig-1.png)

What if we add even more?

The problem will therefore be that our server won't know which node to respond from. If we've added a domain `www.mywebdomain.com` and even if we add a second domain `www.myapidomain.com`, it won't matter, since domains only indicate which IP to direct requests to, and not which port. Therefore, we cannot expect each domain to be answered by different Express servers.

## The Solutions

- One solution would be to contract a server per web application. But this would be a somewhat costly and inefficient solution.

- Another solution could be to specify different ports and have the web servers not listening to the same port. This solution is perhaps more feasible for an API, leaving the React/Angular web on 443 so it can be called from `https://www.mywebdomain.com` (When no port is specified, it goes to 443 if it has https and 80 if it has http) while API requests could be specifically directed to other ports: `https://www.mywebdomain.com:3030`.

  This solution is valid in principle, but if we add more websites that serve frontends, we cannot expect users of those websites to use domains that specify ports. It's a solution that lacks professionalism. Additionally, this solution will bring us other problems when issuing and renewing SSL certificates to use `https://`

- The solution proposed here is to use a type of web server that is the only one listening to those ports 443 and 80, and this in turn forwards those requests to other web servers running locally on the server but on different ports (e.g., 3010, 3011...) and these local servers will not be directly exposed -Their ports will not be open in the firewall-. This type of web server that distributes requests to others is known as a **Reverse Proxy**. If we also combine it with the use of subdomains, we can make many web services available through a single domain: `https://www.mywebdomain.com`, `https://api.mywebdomain.com`, `https://mywebapp2.mywebdomain.com` since the reverse proxy will analyze which domain each request comes from and distribute it to the appropriate local web server.

  It's also a common solution that will work correctly with SSL certificate management clients like _certbot_ and will allow us to automate the process of renewing those certificates.

![Figure 1: A server listening with a reverse proxy to port 443](img/fig-2.png)

## Configuration Process

The process of setting up a web server from scratch goes beyond configuring a reverse proxy and multiple node servers running locally. But not much beyond that. We'll cover the complete process since we all find ourselves having to perform the complete process and it's helpful to have the steps in a single guide.

This process aims to configure a server to be secure and functional for production, so some steps might seem unnecessary, but they are done to improve security.

You might wonder what operating system to have on the server. My recommendation is Ubuntu Server or Rocky Linux (due to their widespread use) in their latest LTS version and WITHOUT desktop environment to avoid wasting resources unnecessarily since we'll perform operations through the terminal.

Before acquiring or contracting the server, whether it's a VPS or a cloud server instance, it's HIGHLY recommended that you're prepared to follow at least the first steps prior to installing web services since once the server is up you can be sure that some automated bot will find your web and will try to login by brute force using default users (like root) and common passwords.

That's why our first steps are aimed at this.

1. Find and analyze the access data we've been given to access our web server.
2. Modify the access data and the possibility of access from known accounts (e.g., like 'root', 'admin', 'administrator'...)
3. Check and Set up the firewall with the necessary exceptions so we can continue connecting to the server.
4. Update --Up to here are the operations to perform urgently as soon as the server is up--
5. Install Node
6. Install NGINX (Our Reverse Proxy)
7. Install MySQL or MongoDB or whatever DB server we're going to use (If we need them)
8. Create a user WITHOUT SUDO PERMISSIONS that will be the owner of the folders where our node servers are contained.
9. Create BASH scripts with commands to start the node servers.
10. Create services that execute these scripts so our node servers function as server services.
11. Configure our domains and subdomains to point to our server.
12. Configure our reverse proxy to direct requests from each domain or subdomain to their corresponding local node servers.
13. Configure certbot to issue and also be in charge of renewing SSL certificates when appropriate.
14. Check everything (Services enabled, Server restart with everything automatically up) and go touch grass.

## 1. FINDING AND ANALYZING THE ACCESS CREDENTIALS PROVIDED FOR OUR WEB SERVER

Once we have completed the contracting process, our server will be available for access within a few minutes. The access credentials can be sent to us via email or obtained through our provider's control panel.

We must analyze the username we've been given. If it's `root`, for example, we have a problem because `root` is a superuser that exists on all Linux machines, but for security reasons is often not given remote access. However, if we've been given access credentials with `root`, it means this user DOES have remote access, and this is problematic because automated bots that try to access servers through brute force are constantly trying combinations with `root` and passwords, some simple like `1234` or `password`, others a bit more complicated, but constantly.

To access through brute force, they need to find your username and password. If they somehow know your username, they already have half the work done. They might, for example, know the IP range of your provider and that access is given with the root user, or other circumstances that let them know your username. That's why it's vitally important that if your provider has given you a too common username, like `root`, `admin` or the provider's name, such as `arsys` or `hostinger` (I'm not saying these providers use these usernames, they're just examples), it's crucial that you either create a new user and remove remote access from the other user.

Then we need to analyze the password, although this should always be changed. In reality, both username and password should be changed. The difference or need for analysis is the urgency with which you must perform these actions. With usernames like `root` or `admin` or `administrator`, the change is extremely urgent. With a common username like the server provider's name, the change is urgent. With a username that looks random like `a12hjdMa`, the change wouldn't be as urgent, but still recommended.

Our provider might allow us to choose the access format, as we can not only access through username and password. We can also access through username and shared key, which we should install on the computer. As this case is less frequent, we'll leave it aside, although it's a more secure method but with the inconvenience of having to install the key on all computers from which access can be made.

Another case we might encounter is that we've been given a specific port for remote connection different from the default port. In that case, congratulations, your server is much more secure just because of this, although now the port is another parameter to remember along with the username and password. But first, let's explain a bit about how we connect remotely for those not familiar with SSH.

### Remote connection through SSH service

To connect remotely to our server, the provider will have set up the server with SSH service enabled. SSH stands for _Secure Shell_ and allows us to connect from any terminal, like `Git Bash`, `CMD`, `PowerShell` or `Terminal Mac`, although we can also use a program like `PuTTy` to make the connection and have the terminal. In any case, to connect via SSH you need to open a terminal program like those mentioned above.

This service has a default connection port: **Port 22**. If our provider doesn't say anything in the access credentials, it means we use that port.
In that case, we could connect to our server using any of the following fictional usernames and addresses:

```Bash
ssh <your username>@<your server address>
```

for example

```Bash
ssh root@122.122.122.122
```

or using a domain name:

```Bash
ssh root@sajkjsd-122.cloud.hostinger.com
```

If the address is correct, it will immediately ask for the password. BE CAREFUL as when typing the password NO CHARACTERS WILL BE SHOWN, NOT EVEN HIDDEN ONES to improve security in case someone could see your screen so they wouldn't even have the data of your password length. This makes it easier to make mistakes, so be careful when entering the password, as you won't be able to see anything to know if you've entered one character too many or too few.

**If our provider has given us a specific port different from 22 for SSH**

Then the connection would be like this, for example, to port 2244:

```Bash
ssh -p 2244 root@122.122.122.122
```

## 2. MODIFY ACCESS CREDENTIALS

To improve our server's security, we'll follow these steps:

#### 2.1 Create a new user with administrator privileges (Sudo)

First, we'll create a new user with _sudo_ privileges. To create a _sudo_ user, our current user must also have _sudo_ privileges. Therefore, we must add the word `sudo` before the command to execute. It may request a `sudo` password when performing the action. It's the same password as the user you used for access. As an exception, the `root` user doesn't need to add the sudo command.

```bash
# Create new user
sudo adduser yournewuser

# Add to sudo group
sudo usermod -aG sudo yournewuser
```

At this point, it's recommended to test if you have SSH access with this new user. Either close the current terminal and open a new one, or use the `exit` command and then use the command again.

#### 2.2 Configure SSH for better security

We'll edit the SSH configuration file using the Nano editor:

```bash
sudo nano /etc/ssh/sshd_config
```

We'll make the following changes in this text file. Find the line that's already written with the parameter name to configure and modify it. DO NOT ADD A NEW LINE WITH THE INSTRUCTION. You must modify the existing line.

```text:/etc/ssh/sshd_config
# Disable root access
PermitRootLogin no

# Disable password authentication (optional, only if using SSH keys - IF NOT IT'S IMPORTANT TO LEAVE IT AS yes)
PasswordAuthentication yes

# Change SSH port (optional but recommended - VERY IMPORTANT - The number is just an example.
# You must find your own port. Just make sure it's available -
# check the list in the link)
Port 2244
```

When changing the SSH port, remember 2 things: Write down the port correctly. Don't use a port that's being used by another service.

[Here's a list of the most common ports to avoid using them if you decide to change the SSH port](https://www.stationx.net/common-ports-cheat-sheet/)

#### 2.3 Restart SSH service

```bash
sudo systemctl restart sshd
```

#### 2.4 Verify access with the new user

Before closing the current session, open a new terminal and verify that you can access with the new user:

```bash
ssh -p 2244 yournewuser@your-server-ip
```

⚠️ **IMPORTANT**:

- Don't close the original session until confirming you can access with the new user
- Save the new SSH port if you changed it
- If you use a firewall, make sure to allow the new SSH port:

```bash
sudo ufw allow 2244/tcp
```

#### 2.5 Disable unnecessary users (optional, but, again, recommended)

If you want to disable users like 'admin' or 'administrator' to improve security and prevent them from connecting via SSH:

```bash
sudo passwd -l username
```

This will lock the account without deleting it.

## 3. CHECK AND SET UP THE FIREWALL

The firewall is an essential part of our server's security. In Ubuntu Server, we'll use UFW (Uncomplicated Firewall), which comes pre-installed but is generally disabled.

### 3.1 Verify firewall status

```bash
sudo ufw status
```

If it appears as "inactive", we'll need to configure it.

### 3.2 Basic firewall configuration

Before activating the firewall, we must ensure we allow SSH connections to avoid being locked out of the server:

```bash
# If using default SSH port (22)
sudo ufw allow 22/tcp

# If we've changed the SSH port (example: 2244)
sudo ufw allow 2244/tcp
```

### 3.3 Enable the firewall

```bash
sudo ufw enable
```

⚠️ **IMPORTANT**: Make sure you've allowed SSH access before enabling the firewall.

### 3.4 Recommended basic rules

```bash
# Deny all incoming traffic by default
sudo ufw default deny incoming

# Allow all outgoing traffic by default
sudo ufw default allow outgoing

# Allow HTTP
sudo ufw allow 80/tcp

# Allow HTTPS
sudo ufw allow 443/tcp
```

### 3.5 Verify configured rules

```bash
sudo ufw status verbose
```

### 3.6 Additional useful commands

**(NOT TO USE NOW, for reference)**

```bash
# Delete a rule
sudo ufw delete allow 80/tcp

# Reload rules
sudo ufw reload

# Disable firewall (not recommended in production)
sudo ufw disable
```

### 3.7 Check if your provider uses another firewall configurable from the control panel

Your provider might use an additional firewall layer. It's good practice to use both firewalls since if the external firewall fails, you won't get any notification and your website would be unprotected.

If your provider uses such external firewall, you'll need to allow both internal and external traffic for the new SSH port and any other service you add. If it's a web server, verify that ports 80 and 443 are open for input and output.

### 3.8 🛟 If you've locked yourself out of the server 🛟

You might have accidentally locked yourself out of the server. Don't despair, it has happened to all of us, and not all is lost.

In this case, you'll need to go back to your provider's control panel and check if they have another type of access called "Panel Access" or "Remote". This would allow us to access through remote desktop as if we were physically in front of the server's monitor, so firewall or SSH configurations wouldn't matter. However, you must know your user and password. From here, we could access and modify the SSH or firewall configurations that have blocked server access.

If there's no such access... then, you'll have to use the control panel option that allows you to reinstall the server and start the process again.

If none of these options are available, contact your provider to have them reset the server with default configurations.

### 3.9 ⚠️ **IMPORTANT NOTES**:

- Always double-check that you've allowed SSH access before enabling the firewall

- If you need to access other services (like databases), remember to open the corresponding ports.

  However, **my recommendation is NOT to open such ports in the case of Databases as long as your API is hosted on the same server** since your database will make a local connection and doesn't need to be exposed to the internet, which is always a risk if unnecessary.

- It's recommended to keep the number of open ports to the minimum necessary

- Consider using allowed IP ranges for critical services if possible
