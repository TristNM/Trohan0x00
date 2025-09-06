---
title: KashiCTF Writeup
published: 2025-02-25
tags: [CTF]
category: Writeups
author: Trohan0x00
draft: false
description: Writeup about KashiCTF
image: "/assets/post_image/kashi.png"
---

# KashiCTF writeup 2025
## I. Corporate Life 1
* Giải này mình vừa chơi chủ nhất tuần trước tiếc là bận quá nên không làm bài nào=))), giờ mới rảnh ngồi viêt wu ahihi
* Bài đầu tiên thì lần đầu làm mình tốn cả ngày làm tìm cách exploit dcmn=)))
![image](https://hackmd.io/_uploads/SyPjtHs9kx.png)
* Nhìn vào đề thì ta cũng hiểu được hoàn cảnh author rồi nên mình sẽ không nói lại nữa. 
![image](https://hackmd.io/_uploads/rJJaKSo5ke.png)
* Ngắm qua route chính ta thấy một bảng, có vẻ là được lấy từ database gồm 3 cột `Employee, Request details và Status`. Theo như đề nói thì có vẻ như `disgruntled employees` đã chặn chúng ta khỏi việc nhìn thấy bí mật của công ty nên do đó ta không thể xem bằng cách thông thường được.
* Đây là chall dạng blackbox, nên ý tưởng ban đầu của mình là sẽ fuzzing route thử xem có thông tin gì thêm không
![image](https://hackmd.io/_uploads/HJsbnBjqkx.png)
* Thử dùng dirsearch thì thấy rằng không có thêm route gì cả, hmm có lẽ do wordlist của dirsearch không đủ route để fuzzing chăng? Sau đấy mình có dùng thử `Burp` thì có thấy được một api là `/api/list`. Truy cập vào thử
![image](https://hackmd.io/_uploads/rk8Y2rj5kg.png)
* Ta nhận được một đoạn json, gồm dữ liệu của các nhân viên ở route chính và có thêm `email` và `role`. Đây có lẽ là phần làm mình tốn thời gian nhất đó là việc đây là một route tĩnh=))), không thể exploit gì khác nên ta có thể bỏ qua role này:V
![image](https://hackmd.io/_uploads/SJPJaroqJg.png)
* Mấu chốt ở chall này là việc nó dùng framework `NextJS`, một framework dựa trên `ReactJS`, và sau một lúc gg tìm cách khai thác route thì ta có được payload [ở đây](https://www.linkedin.com/posts/mandal-saumadip_bugbounty-bugbountytips-activity-7238994256491659264-Poxt/).
* Cách khai thác route ở đây là ta sẽ dùng `console.log(__BUILD_MANIFEST.sortedPages)
` có sẵn của `NextJs` để khai thác route 
    * Trong `Next.js`, `__BUILD_MANIFEST` là một đối tượng `JavaScript` được tạo tự động trong quá trình build của ứng dụng. Đối tượng này chứa thông tin về các trang và các chunk (phần mã) tương ứng, giúp `Next.js` quản lý và tải các tài nguyên một cách hiệu quả.
    * Thuộc tính `sortedPages` trong `__BUILD_MANIFEST` là một mảng chứa danh sách tất cả các đường dẫn (route) của các trang trong ứng dụng, được sắp xếp theo thứ tự xác định
* Cụ thể ta vào console để log ra các route nhờ vào `__BUILD_MANIFEST`
![image](https://hackmd.io/_uploads/BJK1kUo51g.png)
* Ta thấy thêm một route là `/v2-testing`, hmm mình biết vì sao không thể brute được là do chall đã cố tỉnh đặt tên api khác với format thường dùng của các route api trong wordlist:V
* Đến đây mình nghĩ khả năng ta sẽ dùng sqli để khai thác, nhưng sau một hồi inject mình k thể làm gì khác do không thể inject được, check `Burp` thì thấy được web đã call một api khác là `api/list-v2`
![image](https://hackmd.io/_uploads/SJiXxUic1g.png)
* ROute này chỉ nhận `POST request` với param là `filter:` có sẵn. Mình thử inject với payload `or 1=1-- -` thì ta leak được tất cả thông tin từ database
![image](https://hackmd.io/_uploads/BkpebIj9Jx.png)
* đây chính là secret mà ta bị denied ở employee `peter.johson`, có lẽ do mình làm sau khi end giải nên author đã gỡ flag xuống
> KashiCTF{s4m3_old_c0rp0_l1f3_993Y18e3}

## II. Corporate Life 2
![image](https://hackmd.io/_uploads/HkwRM8s51x.png)
* Có lẽ tên phá hoại còn giấu gì đó ngoài flag, để mình xem thử.
* Dưa trên database đã leak ra thì ta thấy rằng gồm 6 column trong bảng mầ câu lệnh select đã trỏ vào. Để chắc chăn hơn mình sẽ dùng `order by 7-- -' để xem server có quăng lỗi không
![image](https://hackmd.io/_uploads/ByukIUi9ye.png)
* Như vậy ta đã biết câu `select` đầu tiên trỏ vào table gồm 6 columns, tiếp đó ta cần biết được server đang dùng loại sql gì, sau một lúc thử các payload thì mình thành công với `"'union select sql,null,null,null,null,null from sqlite_master-- -"`, ta biết được server dùng `SQLITE3` và từ đó biết luôn các bảng được tạo thông qua `sqlite_master`.
![image](https://hackmd.io/_uploads/B1O7w8i9kl.png)
* Ta thấy được 1 column `secret_flag` được tạo ở trong table `flags`, giờ ta chỉ cần select vào và leak data ra
![image](https://hackmd.io/_uploads/SJd0DIoqJl.png)
> KashiCTF{b0r1ng_old_c0rp0_l1f3_am_1_r1gh7_PU4lJNL5}
## III. superfast API
![image](https://hackmd.io/_uploads/B1dC28iqJx.png)
* Theo như đề, đây có vẻ là một web API, giờ mình sẽ dirsearch để fuzzing route 
![image](https://hackmd.io/_uploads/rJJGaUj9kx.png)
* Ta vào `/docs` 
![image](https://hackmd.io/_uploads/rkRh0Uoq1l.png)
* Đây là một route để thực hiện call API, hmm đến đây mình có vẻ nghĩ đến một lỗ hỏng liên quan đến việc call API đó là `Mass Assignment`. Xảy ra khi một `API` hoặc ứng dụng web tự động mapping dữ liệu từ `request` vào `object` trong code mà không validate input. 
* Để thực hiện trước hết mình sẽ thực hiện create user `trohan` vào database
### 1. Create 
![image](https://hackmd.io/_uploads/B142kvj5Jg.png)
* Ta tạo thành công với các credentials cơ bản
### 2. Get user 
![image](https://hackmd.io/_uploads/BkoC1Po9Je.png)
* Khi `Get user` ta thấy user được gán thêm `role : guest`, mấu chốt để khai thác là ở đây, khi ta có thể dùng `PUT user` để có thể cập nhật data và sửa được role của user `trohan` thành `admin`
### 3. Put user
![image](https://hackmd.io/_uploads/SkoExwjq1e.png)
* Thực hiện việc chèn thêm `"role" : "admin"` vào request `PUT user` ta có thể dễ dàng nhận được role mới là `admin`
### 4. GET /flag
* Giờ đây với role `admin` ta dễ dàng truy cập `/flag` để lấy flag
![image](https://hackmd.io/_uploads/rJ3tlvscyg.png)
> KashiCTF{m455_4551gnm3n7_ftw_HABquEXjy}
