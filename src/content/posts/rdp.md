---
title: eJPT Lab Enabling Remote Desktop
published: 2025-09-06
description: Writeup about eJPT Lab RDP
tags: [eJPT, Pentest, Windows]
category: 'Writeup'
draft: false 
image: "/assets/post_image/ejpt.png"
---



# eJPT Labs Windows: Enabling Remote Desktop
> Metasploit provides a large collection of exploits for various software vulnerabilities. These exploits can be used to gain unauthorized access to systems. It also consists of modules that help in maintaining control over compromised systems, collecting sensitive data, and escalating privileges. In this lab, we will explore a couple of modules to exploit a vulnerable application and gain an RDP session on the target.
## Recon
Bước đầu tiên sẽ tiến hành recon với nmap, đối với các lab của `eJPT` thì ip sẽ thường là `eth1 + 1` hoặc được để với domain name là `demo.ine.loca` ở `/etc/hosts` 

```  
Host is up (0.0022s latency).
Not shown: 992 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
80/tcp    open  http         BadBlue httpd 2.7
| http-methods: 
|_  Supported Methods: HEAD
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=9/6%OT=80%CT=1%CU=33843%PV=Y%DS=3%DC=T%G=Y%TM=68BBD
OS:37F%P=x86_64-pc-linux-gnu)SEQ(SP=104%GCD=1%ISR=10D%TI=I%CI=I%II=I%SS=S%T
OS:S=7)OPS(O1=M55FNW8ST11%O2=M55FNW8ST11%O3=M55FNW8NNT11%O4=M55FNW8ST11%O5=
OS:M55FNW8ST11%O6=M55FST11)WIN(W1=2000%W2=2000%W3=2000%W4=2000%W5=2000%W6=2
OS:000)ECN(R=Y%DF=Y%T=7F%W=2000%O=M55FNW8NNS%CC=Y%Q=)T1(R=Y%DF=Y%T=7F%S=O%A
OS:=S+%F=AS%RD=0%Q=)T2(R=Y%DF=Y%T=7F%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T3(R=Y%DF=
OS:Y%T=7F%W=0%S=Z%A=O%F=AR%O=%RD=0%Q=)T4(R=Y%DF=Y%T=7F%W=0%S=A%A=O%F=R%O=%R
OS:D=0%Q=)T5(R=Y%DF=Y%T=7F%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=7F%W=
OS:0%S=A%A=O%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=7F%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U
OS:1(R=Y%DF=N%T=7F%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DF
OS:I=N%T=7F%CD=Z)

Uptime guess: 0.011 days (since Sat Sep  6 11:38:07 2025)
Network Distance: 3 hops
TCP Sequence Prediction: Difficulty=260 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-09-06T06:23:55
|_  start_date: 2025-09-06T06:08:25
| smb2-security-mode: 
|   3:0:2: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

TRACEROUTE (using port 23/tcp)
HOP RTT     ADDRESS
1   0.03 ms ap-southeast-11 (10.10.43.1)
2   ...
3   2.44 ms demo.ine.local (10.0.25.173)

NSE: Script Post-scanning.
Initiating NSE at 11:53
Completed NSE at 11:53, 0.00s elapsed
Initiating NSE at 11:53
Completed NSE at 11:53, 0.00s elapsed
Initiating NSE at 11:53
Completed NSE at 11:53, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 80.46 seconds
           Raw packets sent: 1171 (55.094KB) | Rcvd: 1090 (47.114KB)
```

Ta thấy khá nhiều kết quả, port 80 chạy `BadBlue 2.7` ở version khá cũ khả năng có nhiều lỗ hỏng. Ta còn thấy có các port `135, 49152–49155` chính là các service của `RPC` và port `445` là smb. Mình sẽ tiến hành kiểm tra  web server trước.
![image](https://hackmd.io/_uploads/ryDMY8Yqxg.png)

Web chạy `Badblue 2.7` với tính năng filesearch trên server. Trong msfconsole ta có một module để khai thác version này mình sẽ check thử, ta thấy được module exploit `CVE-2007-6377`. 
![image](https://hackmd.io/_uploads/BJS9xwYcgx.png)

## Exploit & Post-exploit
Mình sẽ run module này và gán `RHOST` với `demo.ine.local`
![image](https://hackmd.io/_uploads/B1o7bwtcxe.png)

Có vẻ như đã lỗi, là do ta đang dùng payload 64bit, `Badblue` là một service cũ chạy 32bit nên cần đổi thành `windows/meterpreter/reverse_tcp`
![image](https://hackmd.io/_uploads/HJqK-PF9ex.png)

Thành công `RCE` check thử quyền thì ta đã có luôn được quyền `system`
![image](https://hackmd.io/_uploads/H1N67vtcel.png)
Giờ là tìm flag, ở đây bài này rất đơn giản nếu ta giải theo cách bình thường là find `flag.txt` ra, nhưng ở đây mình sẽ làm theo cách như đúng hướng của lab muốn đó là bật service `RDP` lên.
Như ta thấy có quyền system nên có thể dễ dàng bật được service tùy thích. Mình sẽ store lại session của shell và tìm module tên `enable_rdp`
![image](https://hackmd.io/_uploads/Hk3HBvFqlg.png)
Giờ chỉ cần set session và run 
![image](https://hackmd.io/_uploads/HJpvHDY5ex.png)
Như ta đã thất `RDP` service đã bật thành công ở port `3389`. Để chắc chắn mình sẽ thử scan target một lần nữa với nmap 
![image](https://hackmd.io/_uploads/S1Mhrwtqgl.png)
Giờ với terminal local mình sẽ connect với phía máy windows bằng `xfreerdp`. Trước đó ta cần có credential để login do không biết được passwd gốc của admin nên mình sẽ trở lại shell và đổi mật khẩu mới 
```bash
xfreerdp /u:Administrator /p:Trohan0x00 /v:demo.ine.local
```
Ta thấy được luôn file flag ở `Desktop`. Và ở cách đầu tiên nếu không bật services `RDP` thì có thể dir file `flag.txt` bằng `dir C:\*flag*.txt /s /b`


> 763e1c86da26c66e86a8537fd343280d  

