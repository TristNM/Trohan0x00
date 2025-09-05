---
title: Cross Site Scripting 
published: 2025-03-10
# image: '/assets/post_image/xss.png'
tags: [Portswigger, Pentest]
category: 'Learning'
image: '/assets/post_image/xss.png'
draft: false 
---


# RESEARCH: Cross Site Scripting (XXS)
![image](https://hackmd.io/_uploads/Byu6VmOmyl.png)
## I. XSS là gì
`Cross-Site Scripting (XSS)` là một lỗ hổng bảo mật phổ biến trong các ứng dụng web. Lỗ hổng này cho phép kẻ tấn công chèn và thực thi mã độc (thường là `JavaScript`) trên trang web mà người dùng khác truy cập.
Các lỗ hổng `Cross-site Scripting` thường cho phép kẻ tấn công giả mạo người dùng nạn nhân, thực hiện bất kỳ hành động nào mà người dùng có thể thực hiện và truy cập bất kỳ dữ liệu nào của người dùng. Nếu người dùng nạn nhân có quyền truy cập đặc quyền trong ứng dụng, thì kẻ tấn công có thể có toàn quyền kiểm soát tất cả các chức năng và dữ liệu của ứng dụng.
### 1. XSS hoạt động như thế nào?
#### a. Kẻ tấn công gửi mã độc (thường là `JavaScript`) vào các điểm đầu vào của ứng dụng web, chẳng hạn:
* Trường bình luận, tìm kiếm.
* `URL` với tham số `GET`.
* Biểu mẫu hoặc `API` đầu vào.
#### b. Ứng dụng web không kiểm tra đầu vào:
Ứng dụng xử lý dữ liệu từ đầu vào mà không mã hóa hoặc lọc các ký tự nguy hiểm (như <, >, ', ").
Mã độc được lưu trữ `Stored XSS` hoặc phản chiếu trực tiếp `Reflected XSS`.
#### c. Mã độc được gửi đến nạn nhân:
Khi người dùng hợp lệ truy cập vào trang web:
* Với `Stored XSS`, mã độc đã được lưu trên máy chủ và sẽ được tải về cùng với nội dung trang.
* Với `Reflected XSS`, mã độc được truyền qua URL hoặc tham số và phản chiếu ngay trong nội dung trang.
#### d. Trình duyệt của nạn nhân thực thi mã độc:
Trình duyệt của nạn nhân tin rằng mã được gửi từ ứng dụng web là hợp lệ và thực thi mã độc.
Kẻ tấn công lợi dụng để:
* Đánh cắp `cookie` hoặc thông tin phiên đăng nhập bằng `document.cookie`.
* Chuyển hướng nạn nhân đến trang web độc hại.
* Thực hiện hành vi giả mạo yêu cầu `(CSRF)`.

### 2. Các loại XSS
Có 3 loại tấn công `XSS` chính thường được sử dụng:
#### `Stored XSS (XSS lưu trữ)`:
Kẻ tấn công chèn mã độc vào dữ liệu được lưu trữ trên máy chủ (như cơ sở dữ liệu).
Khi người dùng khác truy cập vào trang chứa dữ liệu này, mã độc được thực thi.
Ví dụ: Chèn mã `<script>alert('XSS!');</script>` vào một trường bình luận, mã này sẽ chạy khi người khác xem bài viết chứa bình luận đó.
#### `Reflected XSS (XSS phản chiếu)`:
Mã độc được gửi dưới dạng tham số trong URL hoặc biểu mẫu.
Dữ liệu không được kiểm tra đúng cách và được hiển thị trực tiếp trên trang web.
Ví dụ: URL như `http://example.com/search?q=<script>alert('XSS')</script>`.
#### `DOM-based XSS`:
Xảy ra khi mã `JavaScript` trên trình duyệt xử lý dữ liệu đầu vào từ `URL`, DOM mà không kiểm tra đúng cách.
Không cần tương tác với máy chủ, mã độc được thực thi hoàn toàn trên phía client.

## II. Khai thác
### 1. Reflected XSS:
#### Tính chất
Đây là dạng cơ bản nhất của `XSS`. `Reflected XSS` xảy ra khi dữ liệu đầu vào từ người dùng (như tham số URL, dữ liệu gửi từ form, v.v.) được "phản chiếu" ngay lập tức trong phản hồi của server mà không được validate một cách an toàn. Khi nạn nhân truy cập vào một `URL` chứa payload độc hại, mã `JavaScript` sẽ được thực thi trong trình duyệt của họ.
#### Đặc điểm
Đặc điểm:
* Không lưu trữ trên server: `Payload` chỉ tồn tại trong một phiên làm việc (request-response) duy nhất.
* Tấn công theo chuỗi: Kẻ tấn công thường phải gửi link độc hại cho nạn nhân (qua email, tin nhắn, …).
* Phổ biến: Do thường xảy ra ở các ứng dụng web không xử lý dữ liệu đầu vào đúng cách.
#### LABS:
![image](https://hackmd.io/_uploads/r1ldaIl21e.png)

Lab đầu tiên để ta có thể làm quen với `Reflected XSS` nên cũng sẽ vô cùng đơn giản
![image](https://hackmd.io/_uploads/SJg3pUg2kl.png)

Server sẽ là một trang blog, và sẽ có một ô `search box` để ta tìm kiếm, khi nhập thử một chuỗi bất kỳ thì ta nhận thấy kết quả mà ta nhập sẽ được `refer` lại thẳng lên server
![image](https://hackmd.io/_uploads/rkJXCIenkg.png)

Từ đây ta có thể dùng nó để khai thác `XSS`, hãy để ý ở phía `URL`, ta sẽ thấy một param để nhận chuỗi input của ta khi search
![image](https://hackmd.io/_uploads/SJoBA8e3ke.png)

Vậy sẽ như thế nào nếu thay chuỗi nhập bằng một script `Javascript` như thế này
```javascript
<script> alert(1) </script>
```
![image](https://hackmd.io/_uploads/rJuFRUghke.png)

Như ta đã thấy server đã thật sự chạy đoạn script mà ta nhập vào, vì thế cho biết ta đã trigger được lỗ hỏng `XSS` tiềm ẩn trong server. Hacker có thể lợi dụng `URL` này và gửi cho người dùng khác với một script dùng để lấy cookie (Mình sẽ tìm hiểu ở phần sau) nếu ta không validate input một cách sạch sẽ trước khi `refer` thẳng đến server

### 2. Stored XSS
#### Tính chất:
Đây sẽ là dạng lỗ hỏng thứ hai của `XSS`. `Stored XSS` xảy ra khi payload độc hại được lưu trữ trên máy chủ (ví dụ: trong cơ sở dữ liệu, bảng tin, comment, …) và sau đó được phân phối tới nhiều người dùng khi họ truy cập vào nội dung chứa payload đó.
#### Đặc điểm:
* Tính bền vững: Payload độc hại được lưu trữ và sẽ tác động đến nhiều người dùng qua các lần truy cập.
* Nguy cơ cao: Do mã độc được phân phối một cách rộng rãi, nên mức độ tác động thường lớn hơn.
* Ví dụ: Một diễn đàn cho phép người dùng đăng bài mà không kiểm tra nội dung, dẫn đến việc chèn mã `JavaScript` độc hại vào bài viết.
#### LABS:
![image](https://hackmd.io/_uploads/Hko4IPghkg.png)

Lab này ở các bài post, ta sẽ có thêm phần comment
![image](https://hackmd.io/_uploads/H1JA_vg31g.png)

Như cách hoạt động của một lỗ hỏng `Stored XSS`, thì payload ta chèn vào sẽ nằm lại thẳng vào database, ở đây là thông qua việc comment ở các bài post.
![image](https://hackmd.io/_uploads/S1_8FwlnJe.png)

Ta sẽ có payload tương tự ở lab trước và sau đó comment vào bài post này
![image](https://hackmd.io/_uploads/rJH_Fwlh1e.png)

Như ta thấy sau khi đăng cmt thì thẻ `alert` đã lập tức được bật và phần payload khai thác của ta vẫn sẽ nằm lại ở database chờ kẻ xấu số tiếp theo :laughing: 
![image](https://hackmd.io/_uploads/r1R2twln1x.png)

### 3. Dom-based XSS: 
`DOM-based XSS` là dạng tấn công xảy ra hoàn toàn ở phía client. Ở đây, lỗ hổng không phải do server phản hồi mà do cách mà `JavaScript` trên trình duyệt xử lý và thay đổi nội dung của trang (DOM). Nếu dữ liệu không được kiểm tra và làm sạch khi được đưa vào DOM, kẻ tấn công có thể chèn mã độc.
#### Cách kiểm tra lỗ hổng DOM-based Cross-Site Scripting (DOM XSS)
`DOM-based XSS` xảy ra khi dữ liệu đầu vào từ người dùng (như URL, fragment, cookies...) được JavaScript xử lý trên trình duyệt mà không qua kiểm tra đúng cách, dẫn đến việc chèn và thực thi mã độc.
Có hai cách chính để kiểm tra lỗ hổng này:
* Kiểm tra HTML sinks
`HTML sinks` là những vị trí trong `DOM` mà dữ liệu đầu vào có thể bị chèn trực tiếp vào `HTML`, ví dụ như `innerHTML`.
    * Bước kiểm tra:
        * Chèn một chuỗi ngẫu nhiên vào source (ví dụ: location.search).
        * Mở Developer Tools (F12) → Elements, tìm xem chuỗi đó xuất hiện ở đâu.
        * Dùng Ctrl + F để tìm chuỗi trong DOM.
        * Kiểm tra ngữ cảnh xuất hiện của chuỗi đó:
        * Nếu trong thuộc tính HTML `(<tag attr="value">)`, thử chèn dấu nháy kép (") để thoát khỏi giá trị thuộc tính.
        * Nếu trong nội dung `HTML`, thử chèn thẻ `<script></script>` để kiểm tra thực thi `JavaScript`.
    * Chrome, Firefox, Safari sẽ tự động mã hóa URL (location.search, location.hash...), còn IE11 và Edge cũ thì không. Nếu dữ liệu bị mã hóa, việc khai thác XSS có thể bị chặn.
* Kiểm tra JavaScript execution sinks
    * `JavaScript execution sinks` là những vị trí mà dữ liệu đầu vào có thể bị đưa vào và thực thi bởi `JavaScript`, như `eval()`, `setTimeout()`, `document.write()`, `innerHTML`...
    * Bước kiểm tra:
        * Dùng Developer Tools để tìm nơi sử dụng dữ liệu từ nguồn nguy hiểm:
        * Mở Developer Tools (F12).
        * Vào tab Sources → Nhấn Ctrl + Shift + F → Tìm kiếm location, document.URL, window.name...
        * Khi tìm thấy đoạn code liên quan, đặt breakpoint để xem cách dữ liệu được xử lý.
        * Nếu dữ liệu bị gán vào biến khác, tiếp tục truy dấu biến đó để xem nó có bị truyền vào `eval()`, `document.write()` hay không.
        * Khi xác định được sink, thử nhập payload XSS để kiểm tra xem có thể thực thi mã không.
        

#### Labs:
##### Lab: DOM XSS in document.write sink using source location.search
![image](https://hackmd.io/_uploads/r14GbI-hyl.png)

Lab này sẽ dùng `document.write()` ở `Javascript` để thực hiện ghi trực tiếp input thành thẻ `HTML` vào server mà không validate input dẫn đến `Dom XSS`
![image](https://hackmd.io/_uploads/BkttBUbhJe.png)

Như ta thấy, khi nhập bất kì chuỗi nào vào `search box` thì chuỗi input của ta sẽ được đưa vào hàm `trackSearch` sau đó xử lý và được `document.write()` ghi thẳng vào kết quả `searchTerms` mà không hề có việc validate input nào.
Từ đây ta có thể đưa các script vào để thực hiện trigger `XSS`. Nhưng quan sát ở phía trước do thẻ trước đó của ta là thẻ `<img>` nên ta phải thực hiện việc `breakout` khỏi thẻ `img` trước đó
Ta sẽ dùng payload `"><script> alert(1)</script>`. Bằng cách dùng `">` để thực hiện việc đóng thẻ `img` trước đó và sau đó chèn payload `alert` của ta vào
![image](https://hackmd.io/_uploads/r1QyuI-h1x.png)

##### Lab: DOM XSS in document.write sink using source location.search inside a select element
Server sẽ là `E-commerce` và cho ta check stock các sản phẩm 
![image](https://hackmd.io/_uploads/HJHO9Ubnkl.png)

Bật `Devtools` lên thì thấy rằng khi click vào `Check stock` lập tức input của ta là `storeId` sẽ được đưa trực tiếp vào biến `store`
![image](https://hackmd.io/_uploads/SykPj8WnJx.png)

Sau đó dùng `document.write()` để ghi kết quả check stock mà không validate, do đó ta tìm được điểm để trigger `XSS` là ở `storeId`, để khai thác ta cần `break out` khỏi thẻ `Select` ban đầu bao bọc lấy `storeID` của ta trước
![image](https://hackmd.io/_uploads/ByDlh8Z21l.png)

Payload ta sẽ có `">"</select><img src=1 onerror=alert(1)>`
![image](https://hackmd.io/_uploads/ByGNpLb2yg.png)

##### Lab: DOM XSS in innerHTML sink using source location.search
Lab này sẽ khác 2 labs trước là thay vì dùng `document.write` thì ở đây sẽ là dùng `innerHTML` ở sink để đưa kết quả ra server. Vậy thực chất cả hai khác nhau như thế nào?
Nói tóm gọn thì:
* `document.write()` ghi trực tiếp vào trang, nhưng nếu gọi sau khi trang đã tải, nó sẽ xóa toàn bộ nội dung trang.
* `innerHTML` chỉ thay đổi nội dung của một phần tử cụ thể mà không ảnh hưởng đến phần còn lại của trang.
Giờ ta sẽ quan sát thử web, nhập thử một chuỗi vào `search box`
![image](https://hackmd.io/_uploads/HJJFEdZhke.png)

Quan sất thấy, `query` chính là chuỗi ta nhập vào được đưa thằng vào `innerHTML` sau đó in ra thẳng server mà không hề có biện pháp lọc input nào, giờ ta có thể trigger `XSS` một cách đơn giản, nhưng ở `innerHTML` không thể inject bằng thẻ `script` nên ta sẽ inject bằng các thẻ `HTML` ví dụ như là `img` hoặc `iframe`
![image](https://hackmd.io/_uploads/ryeSBu-nkx.png)

## III. XSS Contexts
`XSS contexts`, đề cập đến các phần khác nhau của trang web nơi dữ liệu do người dùng cung cấp có thể được đặt vào. Mỗi bối cảnh có quy tắc riêng về cách dữ liệu được giải thích, và hiểu được chúng là rất quan trọng để phát hiện và ngăn chặn các lỗ hổng `XSS`.
Các Bối cảnh Chính:
* `HTML Context`: Dữ liệu được đặt trực tiếp vào nội dung HTML, dễ bị tiêm các thẻ script như `<script>alert('XSS');</script>`.
* `HTML Attribute Context`: Dữ liệu nằm trong giá trị thuộc tính, như href hoặc src, có thể bị lợi dụng để thêm các trình xử lý sự kiện như `onerror="alert('XSS');"`.
* `JavaScript Context`: Dữ liệu được nhúng vào mã JavaScript, cần phá vỡ chuỗi hoặc ngữ cảnh để thực thi mã, ví dụ '); alert('XSS'); //.
* `URL Context`: Dữ liệu là một phần của URL, có thể bị khai thác bằng cách sử dụng giao thức `javascript:alert('XSS');`.

### 1. XSS Between HTML Tags
#### Lab: Reflected XSS into HTML context with most tags and attributes blocked
Đây sẽ là một bài `Reflected XSS` nhưng hầu hết các thẻ dùng để inject đều bị chặn, hmm đọc đề thì như vâỵ giờ ta sẽ tiến hành xem thử ở server xem
![image](https://hackmd.io/_uploads/HkFfPiZhJg.png)

Khi thử nhập payload đơn giản thì ta nhận được thông báo là `Tag is not allowed`. Server đã filtered, và khi ta thử với các tag khác cũng như vậy. 
Vậy giờ ta nên làm gì? Để tiết kiệm thời gian thay vì ngồi inject thử lần lượt từng các tag thì ta có [cheat sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet) từ `Portswigger` sẽ tiết kiệm thời gian hơn
![image](https://hackmd.io/_uploads/rkVGusWn1g.png)

Như ta thấy sẽ có tất cả các tag và attribute có ích cho ta trong việc pentest lỗ hỏng `XSS` này. Để thực hiện ta bắt đầu bắt request và gửi vào `Intruder` của Burp
![image](https://hackmd.io/_uploads/BygYdibn1x.png)

Ta sẽ tạo sẵn khung `<>` cho tag và chèn `position` vào bên trong như này
![image](https://hackmd.io/_uploads/Syz3ui-2Je.png)

Đi đến `Cheat Sheet` ta copy payload các tags vào và paste vào payload ở `Intruder` và tiến hành `Attack`
![image](https://hackmd.io/_uploads/S1abYi-2kg.png)

Thấy rằng có 2 tags trả về `respone 200`, ta sẽ chú ý ở tag `body`. OK giờ khi đã có tag hợp lệ thì ta cần quan tâm đến event của tag, tương tự như trước ta sẽ copy payload event và dùng `Intruder` để scan
![image](https://hackmd.io/_uploads/BkPhYo-3kl.png)

Có khá nhiều payload trả về `200` nhưng ta có thể trigger `XSS` với event `onresize`. Trong một số trình duyệt, khi nội dung trang web thay đổi hoặc khi trình duyệt thay đổi kích thước, sự kiện `resize` có thể bị kích hoạt. 
Giờ ta tiến hành dùng payload `<body onresize=alert(1)>` để tiến hành trigger 
![image](https://hackmd.io/_uploads/HJYPcjWh1x.png)

Nhận thấy khi ta thay đổi kích thước web, ngay lập tức event `alert` được bật

#### Lab: Reflected XSS into HTML context with all tags blocked except custom ones
Ta tiếp tục scan với cách của lab trước, ta sẽ scan tags truớc
![image](https://hackmd.io/_uploads/r1qucnW3Jg.png)

Ta thấy được một tag khả dụng, y như tiêu đề của chall thì ta có lẽ sẽ có thể sử dụng tag này để trigger `XSS`
![image](https://hackmd.io/_uploads/SyNhq3bn1e.png)

Check thử với server thì ta hoàn toàn không bị filter bởi tag này, mình sẽ thêm các event vào tag ví dụ như `onclick` hoặc `onmouseover` 


Trigger Done! Giờ ta sẽ gửi payload này đến `Exploit Server` của bài

#### Lab: Lab: Reflected XSS with some SVG markup allowed:
Ta thực hiện scan tag bằng `Intruder` trước. Dựa theo đề bài ta target được tag `svg`, ta còn biết thêm được tag `animatetransform`
![image](https://hackmd.io/_uploads/HyakzrEnyx.png)

Giờ ta cần scan event để khai thác, tiếp tục dùng đến `Intruder` ta có được thêm event là `onbegin` từ đây ta có payload để khai thác `<svg><animatetransform onbegin=alert(1)>`

### 2. XSS in HTML tag attributes
#### Lab: Reflected XSS into attribute with angle brackets HTML-encoded
Ta thử nhập một chuỗi ngẫu nhiên và mở `Devtools` lên
![image](https://hackmd.io/_uploads/HkHtRrV2ke.png)

Ta có thể thấy chuỗi ta nhập được đưa vào attribute `value` của thẻ `input`, suy ra ta có thể `break out` thẻ này để tạo thêm một attribute khác, như là `mouseover` chẳng hạn. Ta có script khai thác là `"onmouseover="alert(1)`
![image](https://hackmd.io/_uploads/B1t1kUVhyx.png)

#### Lab: Stored XSS into anchor href attribute with double quotes HTML-encoded
Tại đây, ta có thể thực thi `JavaScript` mà không cần `break out` giá trị attribute. Ví dụ: nếu ngữ cảnh `XSS` nằm trong thuộc tính href của thẻ `a`, thì ta có thể sử dụng giao thức giả `javascript` để thực thi tập lệnh. Chẳng hạn: `<a href="javascript:alert(document.domain)">
`
Giờ đến với lab, sau khi quan sát, ta thấy được phần link website ta nhập vào form được lưu vào thẻ `a`
![image](https://hackmd.io/_uploads/rJTJWUN3kg.png)

Ta có thể dùng payload ở trên để khai thác 
![image](https://hackmd.io/_uploads/SJxNZLV3Jg.png)

#### Lab: Reflected XSS in canonical link tag
![image](https://hackmd.io/_uploads/rJ108L42kl.png)

Ta nhận thấy ở phần thẻ `link` mỗi lần ta thay đổi url hay đổi param cho url thì thẻ `link` đều nối các tham số đó vào url gốc của chall và từ bài [researc](https://portswigger.net/research/xss-in-hidden-input-fields) của `PortSwigger` ta có thể thêm `accesskey` vào để trigger `XSS`, cụ thể payload sẽ là `?%27accesskey=%27x%27onclick=%27alert(1)` và sau khi inject ta nhấn `alt + X` thì lập tức popup sẽ được bậ
![image](https://hackmd.io/_uploads/BJXpwUVhyx.png)

### 3. XSS into Javascript
#### Lab: Reflected XSS into a JavaScript string with single quote and backslash escaped
![image](https://hackmd.io/_uploads/S18n7jr2ke.png)

Ta thấy thằng input ta nhập được đưa vào biến `searchTerms`, và sau đó không có validate gì thêm, dựa theo ngữ cảnh `XSS into Javascript` thì từ đây ta có thể `breakout` bằng cách đóng thẻ `Javascript` này lại và thêm một thẻ mới vào với payload `</script><img src=1 onerror=alert(1)>`
![image](https://hackmd.io/_uploads/r10MNjHhJl.png)

Ta đã thấy giờ hoàn toàn đoạn thẻ `img` của ta thêm vào đã nằm lại trong server và thành một tag `HTML` mới và cũng bật được popup `alert`

#### Lab: Reflected XSS into a JavaScript string with angle brackets HTML encoded
Cách khai thác tương tự lab trước nhưng ở đây thay vì `breakout` thẻ script cũ thì ta sẽ `breakout` giá trị input được truyền vào và sau đó truyền một input mới theo ý của ta vì giá trị thẻ lúc này đã bị mã hóa nên ta không thể `breakout` giống như lab trước.
Cụ thể ta có payload `';alert(1)-'`. Dấu `;` kết thúc chuỗi ban đầu `searchTerms = ''`, cho phép thoát khỏi phạm vi của chuỗi.`alert(1);` thực thi ngay lập tức hộp thoại cảnh báo (XSS).`-''` là một phần tử thừa giúp tránh lỗi cú pháp (JS không báo lỗi nếu sau `alert(1)` có toán tử hợp lệ, dù nó không cần thiết).
![image](https://hackmd.io/_uploads/HkgG5orn1g.png)

#### Lab: Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped
Một số ứng dụng cố gắng ngăn đầu vào thoát ra khỏi chuỗi `JavaScript` bằng cách thoát bất kỳ ký tự ngoặc kép nào bằng dấu `\`. Một dấu `\` trước một ký tự cho trình phân tích cú pháp `JavaScript` biết rằng ký tự đó nên được diễn giải theo nghĩa đen, chứ không phải là một ký tự đặc biệt như dấu chấm dứt chuỗi. Trong tình huống này, các ứng dụng thường mắc sai lầm khi không thoát được ký tự `\`. Điều này có nghĩa là kẻ tấn công có thể sử dụng ký tự `\` của riêng để vô hiệu hóa dấu `\` do ứng dụng thêm vào.
![image](https://hackmd.io/_uploads/rJwcjiB31l.png)

Ở đây mình dùng `\';alert(1)//` để thực hiện `breakout`. Như ở trên thì ta thêm dấu `\` ở trước đoạn cần break và sau đó tiến hành `breakout` thẻ và chèn câu `alert(1)` vào sau đó các phần thừa ở sau mình sẽ chú thích lại bằng `//`

