# Valentine (User & root)

## User

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

## Root

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
