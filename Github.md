# Github Webhooks to Keep Remote Projects in Sync
> Original article see [here](https://www.digitalocean.com/community/tutorials/how-to-use-node-js-and-github-webhooks-to-keep-remote-projects-in-sync).


### Introduction
When working on a project with multiple developers, it can be frustrating when one person pushes to a repository and 
then another begins making changes on an outdated version of the code. Mistakes like these cost time, which makes it 
worthwhile to set up a script to keep your repositories in sync. You can also apply this method in a production 
environment to push hotfixes and other changes quickly. While other solutions exist to complete this specific task, 
writing your own script is a flexible option that leaves room for customization in the future. GitHub lets you 
configure webhooks for your repositories, which are events that send HTTP requests when events happen. For example, 
you can use a webhook to notify you when someone creates a pull request or pushes new code. In this guide you will 
develop a Node.js server that listens for a GitHub webhook notification whenever you or someone else pushes code 
to GitHub. This script will automatically update a repository on a remote server with the most recent version of 
the code, eliminating the need to log in to a server to pull new commits.


### Prerequisites
To complete this tutorial, you will need:
- One Ubuntu 16.04 server set up by following the Ubuntu 16.04 initial server setup guide, including a non-root user 
with `sudo` privileges and a firewall.
- Git installed on your local machine. You can follow the tutorial Contributing to Open Source: 
Getting Started with Git to install and set up Git on your computer.
- Node.js and `npm` installed on the remote server using the official PPA, as explained explained in 
[How To Install Node.js on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-16-04). 
Installing the distro-stable version is sufficient as it provides 
us with the recommended version without any additional configuration.


## Step 1 — Setting Up a Webhook
We’ll start by configuring a webhook for your repository. This step is important because without it, Github doesn’t 
know what events to send when things happen, or where to send them. We’ll create the webhook first, and then create 
the server that will respond to its requests.

Sign in to your GitHub account and navigate to the repository you wish to monitor. Click on the Settings tab in the 
top menu bar on your repository’s page, then click Webhooks in the left navigation menu. Click Add Webhook in the 
right corner and enter your account password if prompted.

- In the Payload URL field, enter `http://your_server_ip:8080`. This is the address and port of the Node.js 
server we’ll write soon.
- Change the Content type to `application/json`. The script we will write will expect JSON data and won’t be able 
to understand other data types.
- For Secret, enter a secret password for this webhook. You’ll use this secret in your Node.js server 
to validate requests and make sure they came from GitHub.
- For Which events would you like to trigger this webhook, select just the push event. We only need the 
push event since that is when code is updated and needs to be synced to our server.
- Select the Active checkbox.
- Review the fields and click Add webhook to create it.

The ping will fail at first, but rest assured your webhook is now configured. Now let’s get the 
repository cloned to the server.


## Step 2 — Cloning the Repository to the Server
Our script can update a repository, but it cannot handle setting up the repository initially, 
so we’ll do that now. Log in to your server:
```bash
ssh sammy@your_server_ip
```

Ensure you’re in your home directory. Then use Git to clone your repository. Be sure to replace `sammy` with your 
GitHub username and `hello_hapi` with the name of your Github project.

```bash
cd ~
git clone https://github.com/sammy/hello_hapi.git
```

This will create a new directory containing your project. You’ll use this directory in the next step.
With your project cloned, you can create the webhook script.


## Step 3 — Creating the Webhook Script
Let’s create our server to listen for those webhook requests from GitHub. We’ll write a Node.js script that launches 
a web server on port 8080. The server will listen for requests from the webhook, verify the secret we specified, 
and pull the latest version of the code from GitHub.

Navigate to your home directory:
```bash
cd ~
```

Create a new directory for your webhook script called `NodeWebhooks`:
```bash
mkdir ~/NodeWebhooks
```

Then navigate to the new directory:
```bash
cd ~/NodeWebhooks
```

Create a new file called webhook.js inside of the NodeWebhooks directory.
```bash
nano webhook.js
```

Add these two lines to the script:
```js
var secret = "your_secret_here";
var repo = "/path/to/you/repo";
```

The first line defines a variable to hold the secret you created in Step 1 which verifies that requests come 
from GitHub. The second line defines a variable that holds the full path to the repository you want to update on 
your local disk. This should point to the repository you checked out in Step 2.

Next, add these lines which import the http and crypto libaries into the script. We’ll use these to create our 
web server and hash the secret so we can compare it with what we receive from GitHub:
```js
let http = require('http');
let crypto = require('crypto');
```

Next, include the child_process library so you can execute shell commands from your script:
```js
const exec = require('child_process').exec;
```

Next, add this code to define a new web server that handles GitHub webhook requests and pulls down the new version 
of the code if it’s an authentic request:
```js
http.createServer(function (req, res) {
    body = '';
    statusCode = 200;
    req.on('data', function(chunk) {
        body += chunk;
    });
    req.on('end', function() {
        let sig = "sha1=" + crypto.createHmac('sha1', secret).update(body.toString()).digest('hex');
        if (req.headers['x-hub-signature'] == sig) {
            if (req.headers['x-github-event'] == 'push') {
                postBody = JSON.parse(body);
                before = postBody.before;
                after = postBody.after;
                ref = postBody.ref;
                cmd = 'cd ' + repo + ' && ' + 'git pull' + ' && ' + 'python' + ' ' + hook + ' ' + 'webhook' + ' ' + before + ' ' + after + ' ' + ref;
                exec(cmd);
            }
            else if (req.headers['x-github-event'] == 'ping') {}
            else {statusCode = 405;}
        }
        else {statusCode = 403;}
        res.setHeader('Content-Type', 'application/json');
        res.statusCode = statusCode;
        res.end();
    });
}).listen(8080);
```

The `http.createServer()` function starts a web server on port `8080` which listens for incoming requests from Github. 
For security purposes, we validate that the secret included in the request matches the one we specified when 
creating the webhook in Step 1. The secret is passed in the `x-hub-signature` header as an SHA1-hashed string, 
so we hash our secret and compare it to what GitHub sends us.

If the request is authentic, we execute a shell command to update our local repository using `git pull`.

The completed script looks like this:
```js
const secret = "your_secret_here";
const repo = "~/your_repo_path_here/";

const hook = ".git/hooks/testbrain-hook.py";
const http = require('http');
const crypto = require('crypto');
const exec = require('child_process').exec;

http.createServer(function (req, res) {
    body = '';
    statusCode = 200;
    req.on('data', function(chunk) {
        body += chunk;
    });
    req.on('end', function() {
        let sig = "sha1=" + crypto.createHmac('sha1', secret).update(body.toString()).digest('hex');
        if (req.headers['x-hub-signature'] == sig) {
            if (req.headers['x-github-event'] == 'push') {
                postBody = JSON.parse(body);
                before = postBody.before;
                after = postBody.after;
                ref = postBody.ref;
                cmd = 'cd ' + repo + ' && ' + 'git pull' + ' && ' + 'python' + ' ' + hook + ' ' + 'webhook' + ' ' + before + ' ' + after + ' ' + ref;
                exec(cmd);
            }
            else if (req.headers['x-github-event'] == 'ping') {}
            else {statusCode = 405;}
        }
        else {statusCode = 403;}
        res.setHeader('Content-Type', 'application/json');
        res.statusCode = statusCode;
        res.end();
    });
}).listen(8080);
```

If you followed the initial server setup guide, you will need to allow this web server to communicate with the 
outside web by allowing traffic on port `8080`:
```bash
sudo ufw allow 8080/tcp
```

Now that our script is in place, let’s make sure that it is working properly.


## Step 4 - Testing the Webhook
We can test our webhook by using `node` to run it in the command line. 
Start the script and leave the process open in your terminal:
```bash
cd ~/NodeWebhooks
nodejs webhook.js
```

Return to your project’s page on Github.com. Click on the *Settings* tab in the top menu bar on your repository’s page, 
followed by clicking *Webhooks* in the left navigation menu. Click *Edit* next to the webhook you set up in Step 1. 
Scroll down until you see the *Recent Deliveries* section.

Press the three dots to the far right to reveal the *Redeliver* button. With the node server running, click *Redeliver* 
to send the request again. Once you confirm you want to send the request, you’ll see a successful response. 
This is indicated by a `200 OK` response code after redelivering the ping.

We can now move on to making sure our script runs in the background and starts at boot. 
Use `CTRL+C` stops the node webhook server.


## Step 5 — Installing the Webhook as a Systemd Service
`systemd` is the task manager Ubuntu uses to control services. We will set up a service that will allow us to start 
our webhook script at boot and use systemd commands to manage it like we would with any other service.

Start by creating a new service file:
```bash
sudo nano /etc/systemd/system/webhook.service
```

Add the following configuration to the service file which tells systemd how to run the script. 
This tells Systemd where to find our node script and describes our service.

Make sure to replace sammy with your username.

```text
[Unit]
Description=Github webhook
After=network.target

[Service]
Environment=NODE_PORT=8080
Type=simple
User=sammy
ExecStart=/usr/bin/nodejs /home/sammy/NodeWebhooks/webhook.js
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable the new service so it starts when the system boots:
```bash
sudo systemctl enable webhook.service
```

Now start the service:
```bash
sudo systemctl start webhook
```

Ensure the service is started:
```bash
sudo systemctl status webhook
```

You’ll see the following output indicating that the service is active:
```bash
● webhook.service - Github webhook
   Loaded: loaded (/etc/systemd/system/webhook.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2018-08-17 19:28:41 UTC; 6s ago
 Main PID: 9912 (nodejs)
    Tasks: 6
   Memory: 7.6M
      CPU: 95ms
   CGroup: /system.slice/webhook.service
           └─9912 /usr/bin/nodejs /home/sammy/NodeWebhooks/webhook.js
```


You are now able to push new commits to your repository and see the changes on your server.

From your desktop machine, clone the repository:
```bash
git clone https://github.com/sammy/hello_hapi.git
```

Make a change to one of the files in the repository. Then commit the file and push your code to GitHub.
```bash
git add index.js
git commit -m "Update index file"
git push origin master
```

The webhook will fire and your changes will appear on your server.

## Step 6 — Installing the githook in git repository
This script is required to send information about changes in the commit to the TestBrain server. 
This script is launched using the webhook.js prepared earlier.

It is required to upload a file received from the TestBrain system to the git server
```bash
scp ~/Downloads/install.py user@server_ip:/path/on/server
```

First you need to install this script into the repository. During installation, you will need to enter the 
full path to the repository. In the case of a regular git repository, you must specify the path to the .git directory.

`Note: Enter path to git repository without add / at the end`

```bash
python install.py install
Enter Full GIT path (/path/to/REPONAME or /path/to/REPONAME/.git):(input) /path/to/repo/.git
```
`ex. Enter Full GIT path (/path/to/REPONAME or /path/to/REPONAME/.git): /home/parallels/remote-hook/.git`

Output
```bash
INFO: GIT path: /home/parallels/remote-hook/.git
INFO: GIT hook copy to: /home/parallels/remote-hook/.git/hooks/testbrain-hook.py
INFO: GIT hook copied.
INFO: GIT hook wrapper generated.
INFO: Install successfully.
INFO: RUN 'cd /home/parallels/remote-hook/; python /home/parallels/remote-hook/.git/hooks/testbrain-hook.py sync'
```

If the repository already has data, you need to go to the directory with the repository and start synchronization.
```bash
cd /path/to/repo/; python /path/to/repo/.git/hooks/testbrain-hook.py sync

```

Example output
```bash
[XXX] refs/heads/master
[YYY] refs/heads/master 3
[200] refs/heads/master 3
--- Total 30.1865971088 seconds
```


## Conclusion
You have set up a Node.js script which will automatically deploy new commits to a remote repository. 
You can use this process to set up additional repositories that you’d like to monitor. 
You could even configure it to deploy a website or application to production when you push your repository.

