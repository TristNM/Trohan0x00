---
title: Cron Jobs Gone Wild II in Linux Privilege Escalation
published: 2025-05-30
image: '/assets/post_image/cron.png'
tags: [eJPT, Privilege Escalation, Linux]
category: 'Learning'
draft: false 
---



# Cron Jobs Gone Wild II in Linux Privilege Escalation
## I. Crone Jobs là gì?
### 1. Khái niệm
![image](https://hackmd.io/_uploads/SkguEMbC-ex.png)
Dạo gần đây mình có học được một cách thức để có thể leo quyền trên hệ thống Linux khá hay và mình sẽ chia sẻ lại cho mn!

Cách thức này sẽ dựa vào việc `Misconfiguration` ở các file shellcript được thực hiện bởi `Crone Jobs`. Vậy `Crone jobs` là gì?
`Crone jobs` là chương trình để xử lý các tác vụ lặp đi lặp lại ở lần sau. Cron Job đưa ra một lệnh để lên lịch “làm việc” cho một hành động cụ thể, tại một thời điểm cụ thể mà cần lặp đi lặp lại. 
Hãy tưởng tượng ta có một người trợ lý cần mẫn, không bao giờ quên việc. Ta chỉ cần đưa cho người trợ lý đó một danh sách công việc cùng với lịch trình chính xác đến từng phút, và người đó sẽ tự động thực hiện chúng mà không cần bạn phải nhắc nhở.
`Cron` chính là "người trợ lý" đó. Nó là một trình nền (daemon) chạy liên tục trên hệ thống của bạn.
`Cronjob` là một "công việc" hay một "lệnh" cụ thể mà ta giao cho Cron thực hiện theo một lịch trình định sẵn.
Nói một cách kỹ thuật, Cron là một trình lập lịch công việc dựa trên thời gian, và Cronjob là một tác vụ được lên lịch bởi Cron.

### 2. Cách sử dụng
Ví dụ mình có một file gọi là `cleaning.sh`, file này sẽ chứa `Bash script` để có thể tự động clean các file `temp` ở `/ect/temp` và mình đặt chu kỳ dọn dẹp là mỗi 1 tuần. Vậy nên mình sẽ tự động hóa việc này bằng cách dùng `Crone Jobs`.

Đầu tiên mình sẽ sử dụng lệnh `cronetab -e` để tạo một cronetab
Ta có cấu trúc của một lệnh crontab như sau
```
┌───────────── phút (0 - 59)
│ ┌───────────── giờ (0 - 23)
│ │ ┌───────────── ngày trong tháng (1 - 31)
│ │ │ ┌───────────── tháng (1 - 12)
│ │ │ │ ┌───────────── ngày trong tuần (0 - 7) (0 và 7 đều là Chủ Nhật)
│ │ │ │ │
* * * * * /path/to/command-to-execute
```
* Ví dụ: 0 3 * * *  /script/cleaning.sh

Ngoài ra ta còn có một vài`short-hand` phổ biến nhử `@daily, @weekly, @monthly,...`

-> Khi hoàn thành ví dụ ta thiết lập hằng tuần thì mỗi tuần `Crone` sẽ thực thi file trong `cleaning.sh` một lần để thực hiện việc tự động hóa này

## II. Privilege Escalation throug Crone Jobs
### 1. Nguyên nhân
Ta thấy tiện lợi như thế, nhưng nếu trong việc cấu hình không đúng thì sẽ dễ dẫn đến lỗ hỏng cho các Attacker có thể lợi dụng để leo quyền.
* Ví dụ: Nếu tương tự như ở trên mình đã cho ví dụ, khi người dùng tự động hóa bằng `Crone` nhưng ở đây lại là `Root user` vì thế nếu ta để cho `Crone` thực thi dưới quyền root thì sẽ tiềm ẩn nhiều nguy cơ bị leo quyền. Nếu ta Attacker là người dùng bình thường không có quyền thực thi hay đọc các tệp, nhưng lợi dụng vào `Crone` thì khả năng có thể từ đấy lợi dụng `Crone` để có thể cấp quyền

### 2. Lab
Mình cũng không biết giải thích bằng miệng dễ hiểu như nào nên mình sẽ có một lab đơn giản để ta có thể khai thác và hiểu hơn về lỗ hỏng leo thang đạc quyền này

Ta được cấp một shell, giả sử đây là `Linux server` khi ta đã có thể `RCE` vào mục tiêu, nhưng quyền hạn ở đây rất hạn chế nên vì vậy ta cần tìm cách để leo quyền lên `Root user`
![image](https://hackmd.io/_uploads/Bk0EcbCWlx.png)

Mình thử dùng `ls` thì thấy được một tệp `message` nhưng khi chạy thì lại không có quyền để thực thi
![image](https://hackmd.io/_uploads/r1bYqbR-lx.png)

Mình quyết định tìm xem còn file nào có tồn tại trên hệ thống nữa không thì thấy một file trùng tên nằm ở `/tmp/message`
![image](https://hackmd.io/_uploads/B1mdo-C-lg.png)

Từ đây ta có thể thấy được, file này khả năng được copy từ `/home/student` đến `/tmp/` và ở đây ta có thể đọc được nội dung bên trong, không bị chặn permission như ở `/home/student`. Tức là khi được `Crone` sao chép nội dung từ `/home/student` đến `/tmp/` thì nội dung được sao chép nhưng quyền thì lại không được sao chép theo thay vào đó, nó được quyết định bởi một thiết lập hệ thống gọi là `umask` (user file creation mask) của người dùng đang chạy lệnh đó (là root).
![image](https://hackmd.io/_uploads/ryupnbAblg.png)

Giờ ta đã xác định được hệ thống bị lỗ hỏng `Crone Jobs`, giờ ta sẽ tìm ra file thực thi lệnh copy(file .sh). Mình sẽ dùng lệnh `grep -nri "/tmp/message" /`, Vì để thực thi việc copy thì nội dung trong file `.sh` phải đưa được path vào nên ta sẽ tìm nội dung trong file để có thể tìm ra được file `.sh`
![image](https://hackmd.io/_uploads/H1l5z0bRWle.png)

Mình thấy được file `.sh` của ta chính là `copy.sh` nằm ở `/usr/local/share/` giờ ta sẽ cd đến đó và đương nhiên ta hoàn toàn có thể đọc được file thực thi bên trong
![image](https://hackmd.io/_uploads/H1VKCbRbxl.png)

Giờ việc của ta là chỉnh sửa lại permission cho student để leo lên thành `Root user` bằng cách dùng
```bash
printf '#! /bin/bash\necho "student ALL=NOPASSWD:ALL" >> /etc/sudoers' > /usr/local/share/copy.sh
```
Do khả năng cao là shell sẽ không cho user quyền dùng nano hay vim hoặc bất cứ gì sửa đổi khác nên ta sẽ dùng `printf`. Còn bây giờ ta sẽ chờ cho `Crone Jobs` đến chu kỳ mới để copy file ta vừa sửa và chạy nó
![image](https://hackmd.io/_uploads/rkgO1MRWle.png)

Như ta đã biết, việc chạy một chu kỳ mới sẽ hoàn toàn do người tạo quyết định nên thực tế sẽ có thể mất khá lâu, nhưng ở LAB mình đã set thành 1 phút cho chu kỳ mới nên ta không phải đợi lâu
![image](https://hackmd.io/_uploads/SyJTyzRZll.png)

Như đã thấy ta có thể truy cập sudo để vào root và check quyền thì ta hoàn toàn tự do trên shell:v

--> Vậy yếu tố tiên quyết là có một `Cron Job` đang chạy với quyền root. Đây chính là "phần thưởng" mà kẻ tấn công nhắm tới. Nếu tác vụ đó chỉ chạy với quyền của người dùng thông thường (student hoặc devuser), thì dù kẻ tấn công có chiếm được quyền thực thi script, họ cũng không thể làm gì hơn ngoài những việc mà họ vốn đã có thể làm. Việc chạy với quyền root đã tạo ra một tiềm năng gây sát thương cực lớn.
