# 100% VPS-Hosted/Self-Hosted SMS App

## How to install/Deploy https://github.com/0perationPrivacy/VoIP in an LXC container

---


## Index 
- [Introduction](#introduction)
- [Set up your Ubuntu Server](#set-up-your-ubuntu-server)
-  [Create a Sub user](#create-a-sub-user)
-  [Create SSH keys for each user](#create-ssh-keys-for-each-user-and-send-the-keys-to-the-server)
-  [Create SSH profiles for each user](#create-ssh-profiles-for-each-user)
-  [Set up two factor authentication](#set-up-two-factor-authentication)
-  [Require three forms of ID](#require-three-forms-of-id)
- [Prepare LXC container](#prepare-lxc-container)
- [Install mongodb](#install-latest-mongodb)
- [Secure mongodb](#secure-mongodb)
- [Clone repo & build](#clone-voip-repo--build)
- [Configure repo](#configure-repo)
- [Configure Nginx](#nginx-reverse-proxy)
- [Configure the Firewall](#configure-the-firewall)
- [Configure Certbot](#configure-certbot)
- [Install Certbot](#install-certbot)
- [Get your certificates](#get-your-certificates)
- [Configure Telnyx](#configure-telnyx)
- [Run it](#run-it)
- [Last Minute Notes](#notes)

## <a href="intro">Introduction</a>
This guide is **NOT** a fork of OperationPrivacys' program. It is meerly an alternative option for those that seek utmost privacy at all costs. Original credit for is due to @[timconsidine](https://github.com/timconsidine) for [this guide](https://github.com/timconsidine/LXC-OperationPrivacy-VOIP). We have only added to it.

*Needed Disclaimer: Fight For Privacy, LLC strongly believes in owning 110% of your data. Always question who is your data maintainer and what their motive is. We are a bunch of nerds who enjoy security, privacy, self-hosting, and privacy. While we stand fervently by everything we say, nothing is 100% **hack proof**. You should never expect something to be. We will provide a minimum level of protection that you should take, but we actively encourage you to seek additional security information from every reputible source you can find. In addition, you should never store anything online without one or more layers of encryption depending on the sensitivity of the content. We are not responsible for anything you do.*

This guide assumes the following:
- You just obtained this server
- This machine is running a fresh Ubuntu 20.04, or latest Ubuntu OS
- You have root access
- Your privacy matters enough to you to drop a few dollars to protect it

This guide will use the following
- LXC
- MongoDB
- GIT
- Nginx
- Telnyx
- Certbot
- Linux CLI
- and more. 

tl;dr 
- @[0perationPrivacy](https://github.com/0perationPrivacy) Created an awesome tool that @[timconsidine](https://github.com/timconsidine) created [this guide](https://github.com/timconsidine/LXC-OperationPrivacy-VOIP) to self-host for. We made our own guide with more detail.
- Figure out what [LXC is](https://linuxcontainers.org/lxc/introduction/)
- Fight For Privacy, LLC isn't liable if you follow this guide

---
## <a href="setup-ubuntu">Set up your Ubuntu Server</a>

### <a href="subuser">Create a Sub user</a>

We will be creating a subuser called `second`, but feel free to call your subuser whatever you want. To create the subuser, run these commands in order

`adduser sammy`
`usermod -aG sudo sammy`

These two commands create the subuser and then adds the subuser to the sudo user group

### <a href="createkeys">Create SSH keys for each user, and send the keys to the server</a>

Start by creating the root key by running `ssh-keygen -f ~/.ssh/root_rsa -b 4096`, make sure to set a good passphrase you *will not* forget. Once you have that key created, run this command `ssh-copy-id root@remote.server.ip.address`. You will likely see a message similar to 

> Are you sure you want to continue connecting (yes/no)?

If you do, type `yes` and hit enter. When prompted, type in the password for root. If you were successful, you will see

> Output
> Number of key(s) added: 1


Now you are going to create a second key by running `ssh-keygen -f ~/.ssh/second_rsa -b 4096`. This key will be for the subuser created above. Once again, use a different, but memorable passphrase. You will also run the command `ssh-copy-id second@remote.server.ip.address`, type in the subusers password, and if successfull should see

> Output
> Number of key(s) added: 1


### <a href="sshprofile">Create SSH profiles for each user</a>

Next you will create an SSH config profile on your local computer. These are helpful when you have multiple SSH keys or have an SSH key that doesn't have the default naming scheme of id_rsa. Since you now have two SSH keys, you need to tell you machine where to look for the right SSH key. To do this, you will create an [SSH Config file](https://linuxize.com/post/using-the-ssh-config-file/). Just a note that this file is a per-user configuration, it is not system wide. 

To create your users ssh config file, run this command `nano ~/.ssh/config`. You can also create this file manually in MS Windows (Ew) - go to C:\Users\<user_profile>\.ssh, or if using Linux (Yay!), go to /home/<user_name>/.ssh

If you do not have an existing config file, pase the following into the file

Host rootSMS
	HostName remote.server.ip.address
	Port 22
	User root
	IdentityFile ~/.ssh/root_rsa
	IdentitiesOnly yes

Host secondSMS
	HostName remote.server.ip.address
	Port 22 
	User second
	IdentityFile ~/.ssh/second_rsa
	IdentitiesOnly yes

Host # Nickname of connection
	HostName # Enter the remote IP address
	Port # Defaults at 22. Refer to [this article](https://linuxize.com/post/how-to-change-ssh-port-in-linux/) to change your ssh port
	User # The username used to login with
	IdentityFile # Location of the SSH public key
	IdentitiesOnly yes # Requires the use of the identityFile. Prevents errors.

With this now complete, you can open two terminals and run `ssh rootSMS` in one and `ssh secondSMS` in the other. This will establish an SSH connection to both accounts into your remote server. 
Once connected, in each terminal run `chmod -R go= ~/.ssh`. This recursively removes all “group” and “other” permissions from the ~/.ssh directory. 

### <a href="setup_2fa">Set up two factor authentication</a>

Now that you have a session open for both accounts, you will set up two factor authentication using the `libpam-google-authenticator` module. 

Run these commands in the root session in order

> sudo apt-get update

> sudo apt-get install libpam-google-authenticator

Then in the root session, run

> google-authenticator

In order, we reccomend you select these options. `y`, `y`, `y`, `n`, `y`

### <a href="force_2fa">Require the use of 2FA</a>

Since you will be making changes over SSH, it’s important never to close your initial SSH connection. Instead, open a second SSH session test with. This will prevent locking yourself out of your server if there was a misconfiguration.

Start by making a backup of the `SSHD` configuration file

> `sudo cp /etc/pam.d/sshd /etc/pam.d/sshd.bak`

Then open the file 

> `sudo nano /etc/pam.d/sshd`

At the bottom below `@include common-password`, add 

> auth required pam_google_authenticator.so

On your keyboard hit `ctrl + o` and `ctrl + x`

To ensure that SSH supports 2fa, you will need to edit the sshd_config. First, back the file up by running `sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak` then open then file by running `sudo nano /etc/ssh/sshd_config`. Within this file look for `ChallengeResponseAuthentication` and set its value to `yes`. 

Once done, restart the SSH service by running `sudo systemctl restart sshd.service`

**Important** Test that everything worked by opening another session for both users on your server by running `ssh rootsms` and `ssh secondssh`

At this point, you will not need to use your password to access the server on either new session. To ensure only someone using your user account on your computer, using your password and your 2FA device can access your server, follow the next section.

### <a href="three_forms">Require three forms of ID</a>

To set up the requirement to use 2FA, SSH, and your password run this command `sudo nano /etc/ssh/sshd_config` and paste `AuthenticationMethods publickey,password publickey,keyboard-interactive` at the bottom of the file. Then On your keyboard hit `ctrl + o` and `ctrl + x`. Then run `sudo systemctl restart sshd.service`

At this point to ensure everything is working as it should, run `ssh rootsms -v` and `ssh secondsms -v`. The `-v` switch means verbose, so you will see output similar to 

> ```
> debug1: Authentications that can continue: publickey
> debug1: Next authentication method: publickey
> debug1: Offering RSA public key: /Users/<sample>/.ssh/id_rsa
> debug1: Server accepts key: pkalg rsa-sha2-512 blen 279
> Authenticated with partial success.
> debug1: Authentications that can continue: password,keyboard-interactive
> debug1: Next authentication method: keyboard-interactive
> Verification code:
> ```

---
## <a href="prep">Prepare LXC Container</a>

`lxc --version` : confirms that LXC is installed on your Linux instance.

**IF THIS IS YOUR FIRST USE OF LXC**, you need to first run `lxd init`.  There will be a bunch of questions, most of which can be answered using the default option (press `enter`).  I suggest you follow https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-lxd-on-ubuntu-20-04 for the detail.

If you have done `lxd init`, then these are the basic steps you need to take to fire up a container.

`lxc launch ubuntu:20.04 <containername>` : creates a new Ubuntu 20.04 container with your chosen name.  First time may take a few minutes to download the OS image.  

`lxc exec <containername> -- /bin/bash` : enter the container as root (similar to normal Linux su user).  Note shell prompt should change.
 
`hostname` : should confirm you're in the right place
  
`apt-get update && apt-get upgrade -y` : ensures latest packages 

`apt autoremove -y` : tidy up
  
---
## <a href="install-mongo">Install Latest MongoDB</a>
Ubuntu has an old version of mongo db (v3.x).  You probably want the latest version.
For full details, please read the excellent guide at https://www.digitalocean.com/community/tutorials/how-to-install-mongodb-on-ubuntu-20-04
 
`curl -fsSL https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -`
 
`echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list`
 
`apt-get update -y`
 
`apt install mongodb-org -y`
 
`systemctl start mongod.service`

`systemctl status mongod`
 
That should show a running mongo instance.  If not, check the detailed guide linked above.
 
`systemctl enable mongod`
 
`mongo --eval 'db.runCommand({ connectionStatus: 1 })'`
 
`systemctl status mongod`

---
## <a href="secure-mongo">secure mongodb</a>
You probably want a secure mongo db.
The following is an abstract of excellent guide at https://www.digitalocean.com/community/tutorials/how-to-secure-mongodb-on-ubuntu-20-04

 `mongo` : enters interactive shell - enter these commands line by line
  ```
	show dbs
	use admin
	db.createUser(
	{
	user: "mongoadmin",
	pwd: passwordPrompt(),
	roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
	}
	)
 ```
Closing bracket complete the command : you should be prompted for your password and then get a confirmation message that user created successfully.

Then `exit` the mongo shell and return to the container command line 


`nano /etc/mongod.conf`. : <blatant plug : use `ne` editor, it's much nicer, `apt install ne -y`>

Find the line which says `#security` and remove the hash symbol

Add a new line below it with `authorization: enabled` BUT INDENT 2 spaces

Note the formatting :
```
# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

security:
  authorization: enabled

#operationProfiling:

```
save the file and quit it
 
`systemctl restart mongod`
 
`systemctl status mongod`
 
`mongo -u mongoadmin -p --authenticationDatabase admin`
 
`show dbs` : confirms you can see databases currently installed (nothing much) 

 `exit`

---
## <a href="clone-repo">clone VoIP repo & build</a>
Now we can get going ...

`git clone https://github.com/0perationPrivacy/VoIP/`

`cd VoIP`
 
`apt install nodejs -y`
 
`apt install npm -y`
 
`npm install`

---
## <a href="configure-repo">Configure Repo</a>
The following is not particularly clear (at time of writing anyway) from the official guides.
	
And it is the magic which makes it all work. 

`nano .env` or use your favourite editor (cough : `ne`)

ensure that the following is set up (not specified in the repo sample .env) : you can leave the other lines
```
DB = mongodb://<db_admin_name>:<db_password>@localhost/admin
BASE_URL = https://<domain>.<tld>/
```

NB : as per Troubleshooting, ensure the `BASE_URL` has protocol (https) not just url, and ensure it has trailing `/`

---> Now `exit` from your container back to host VPS <---

---
## <a href="nginx">nginx reverse proxy</a>
- NB : this is done in your host VPS : check you are not still in your container
- set up your DNS to point your domain or subdomain to your VPS
- your container is bridged auto-magically when the container is built and launched
- but that's only fine for outbound traffic : you need inbound traffic (doh!) as well
- your installation may be different but in my case I need `nginx` running on the VPS and then a `nginx` conf file to direct traffic to the container
- do `lxc list` to get the local ip address of your container : you need it for the nginx conf file
- create the conf file :  `nano /etc/nginx/sites-available/<domain>.<tld>`
- sample nginx conf file below pre-SSL enabling
- enable it : `ln -s /etc/nginx/sites-available/<domain>.<tld> /etc/nginx/sites-enabled/<domain>.<tld> `
- check config : `nginx -t`
- reload config : `systemctl reload nginx`

sample nginx conf file as BEFORE you do your certbot dance to get https enabled :
```
server {
       listen 80;
       listen [::]:80;

       server_name <domain>.<tld>;

       location / {
                proxy_pass http://<container_ip>:3000;
                proxy_http_version 1.1;
                proxy_cache_bypass $http_upgrade;
        # Proxy headers
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        "upgrade";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  $server_port;
       }
}
```

## <a href="firewall">Configure the Firewall</a>
### For an indepth overview of ufw, see this [article](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands)

Run the following commands

> sudo ufw allow ssh 
> sudo ufw allow https
> sudo ufw allow 'Nginx Full'
> sudo ufw allow 80
> ufw enable
> ufw status 

You should see something similar to the following

	![image](https://user-images.githubusercontent.com/92276223/140882255-5ff86d3c-6b0e-4691-9887-fb5056b17b46.png)


> Stop and ensure you can SSH in from another terminal before continuing. If you can't, solve the problem in your old terminal window before closing it out. Otherwise, you will lose access to your server.

## <a href="certbot">Configure Certbot</a>
If your using https:// in your `.env` in the BASE_URL

### <a href="install_certbot">Install Certbot</a>

> sudo add-apt-repository ppa:certbot/certbot

> sudo apt update

> sudo apt install python-certbot-nginx

Check your nginx config

> sudo nginx -t

If there are no errors, great! If there were, check for spelling errors. Once fully resolved, reload nginx

> sudo systemctl reload nginx

### <a href="get_certbot">Get your certificates</a>

For every domain you need a certificate for (app.com, my.app.com, your.app.com, the.app.com, etcetera) use the format `-d sub.domain.com`

> sudo certbot --nginx -d sub.domain.com

This tells Certbot to use the -nginx plugin, the -d tag tells Certbot which domains the certificate should be valid for.

- You will then be prompted for an email address, to agree to Terms of Service, and ask if you want to subscribe to EFF's newsletter. The first two are mandatory. 
- Then for most sites choose `2: Redirect`

Now we will want to confirm that there are no issues for future issuance of certificates. Run this command

> sudo certbot renew --dry-run

If it runs successfully, then you have nothing to worry about! 

---
## <a href="telnyx">Configure Telnyx</a>
Telnyx costs around $1.10 usd/month per number 
You can register for an account using this [link](https://refer.telnyx.com/j3lki). It will give you $20 in free credits after you verify and set up your account.

But note the following :
- Pick a number within a region of your choosing and make sure it is SMS, MMS, Voice, and Fax enabled 
- you need to Get at least L1 verified to use it, and you may well need L2 verification if you're not in US
- L2 verification needs 24-48 hours, but if you leave it say 12 hours and talk by chat very nicely to support, you may be lucky and they do it then & there.
- see also Troubleshooting notes in OperationPrivacy/VoIP wiki

---
## <a href="run">Run It</a>!
You should be all set to fire it up.

Go back into your container : `lxc exec <containername> -- /bin/bash`

`cd VoIP`
	
`node app.js &`

Then visit your installation URL, sign up for an account (your own), and add a "Profile" which configures the service provider (Telnyx or Twilio) and the number you will be using for SMS.

You will need the [API key](https://portal.telnyx.com/#/app/api-keys) from your Telnyx account, and it should show your number(s) available.
	
Test it.

If you need to stop the app, the brutal way is `killall node`.  Because you're running in a LXC container, you should not affect anything else.

If you leave the terminal running, you can see status messages when you interact with the browser.

If you ensure you use the trailing `&`, you can exit from the container and your VPS terminal, and the app should continue running.

NB : if you later edit the `.env` to turn off signups (so others cannot hijack and abuse your installation), you will need to kill and restart node, and also delete the profile in the app and set it up again : see the Notes section below.

---
## <a href="notes">Notes</a>
- **the big one** : config changes in the app, e.g. the `.env` file, need the profile in the browser to be deleted and re-created.  This will lead to past messages being lost.  Save them somehow if you need them. **As long as the .env file contents are the same, you can delete the entire repo, and paste in saved copy of the .env file** Accurrate as of 11/08/21
- Outbound voice works. However, there is no dial pad. So you can not "Press 1 for ..." and so on.
- Image attachments work
- Telnyx has rate limits to avoid sending abuse
- The app seems to intermittently go down for 3-5 minutes at a time and then come back up. This is generally due to not leaving the LXC container, and then exiting the SSH shell properly. If you run the app with `lxc exec <containername> -- /bin/bash && cd VoIP && node app.js &` then wait for "Database Connected Successfully" to appear, and on the keyboard select Ctrl+A+D to exit the container then leave the SSH shell, you should have a 99.99999% uptime. 

I haven't tested any keepalive measure to ensure the app stays running (check out notes at bottom of Michael's instructions).  I am not convinced they are needed but hey, what do I know. A self hosted "Uptime Kuma" instance is more then enough to know when it goes down, and all you need to do to get it back up is, in order, 
1. SSH into your server
2. run `lxc exec <containername> -- /bin/bash && cd VoIP && node app.js &` 
3. Wait for "Database Connected Successfully" to appear
4. Select Ctrl+A+D to exit the container then leave the SSH shell

I hope this is helpful.
Let me know any corrections, omissions or questions.
                                    
