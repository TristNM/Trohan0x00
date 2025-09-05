---
title: NoSQL Injection
published: 2024-10-10
description: Learning about NoSQL Injection
tags: [Portswigger, Pentest]
category: 'Learning'
draft: false 
image: "/assets/post_image/nosql.png"
---

# NoSQL Injection (NoSQLI)
## I. NoSQL Injection là gì?
### 1. NoSQL là gì?
![image](https://hackmd.io/_uploads/BkOmiyHyJg.png)

* Trước hết ta cần biết `NoSQL` là gì và sự khác biệt với `SQL`. 
* `NoSQL (Not Only SQL)` là một loại hệ quản trị cơ sở dữ liệu không dựa trên mô hình quan hệ truyền thống như `SQL`. Thay vào đó, `NoSQL` hỗ trợ các loại cơ sở dữ liệu có cấu trúc dữ liệu khác như `document, key-value, wide-column, và graph`. Sự khác biệt chính giữa `NoSQL` và `SQL` là cách chúng lưu trữ và truy xuất dữ liệu.
* Có rất nhiều loại cơ sở dữ liệu `NoSQL` khác nhau. Để phát hiện các lỗ hổng trong một cơ sở dữ liệu `NoSQL`, điều quan trọng là hiểu rõ mô hình và ngôn ngữ của chúng.
* Một số loại cơ sở dữ liệu NoSQL phổ biến bao gồm:
    * `Document stores` - Lưu trữ dữ liệu trong các tài liệu bán cấu trúc, linh hoạt. Chúng thường sử dụng các định dạng như `JSON, BSON và XML`, và được truy vấn qua API hoặc ngôn ngữ truy vấn. Ví dụ bao gồm `MongoDB và Couchbase`.
    * `Key-value stores` - Lưu trữ dữ liệu theo định dạng khóa-giá trị. Mỗi trường dữ liệu được liên kết với một chuỗi khóa duy nhất. Giá trị sẽ được truy xuất dựa trên khóa này. Ví dụ bao gồm `Redis và Amazon DynamoDB`.
    * `Wide-column stores` - Tổ chức dữ liệu liên quan vào các cột linh hoạt thay vì các hàng truyền thống. Ví dụ bao gồm `Apache Cassandra và Apache HBase`.
    * `Graph databases` - Sử dụng các node để lưu trữ các thực thể dữ liệu, và các cạnh (edges) để lưu trữ mối quan hệ giữa các thực thể. Ví dụ bao gồm `Neo4j và Amazon Neptune`.

    | Đặc điểm                   | SQL                                      | NoSQL                                   |
    |----------------------------|------------------------------------------|-----------------------------------------|
    | Kiểu cơ sở dữ liệu          | Cơ sở dữ liệu quan hệ (`Relational DB`)    | Cơ sở dữ liệu phi quan hệ (`Non-relational DB`) |
    | Cấu trúc dữ liệu            | Bảng với hàng và cột                    | `Document`, `Key-Value`, `Wide-Column`, `Graph` |
    | Mô hình quan hệ             | Có, sử dụng `khóa chính` và `khóa ngoại` | Không có hoặc hạn chế quan hệ giữa dữ liệu |
    | Ngôn ngữ truy vấn           | `SQL` (Structured Query Language)        | API hoặc ngôn ngữ truy vấn riêng tùy loại |
    | Khả năng mở rộng            | Mở rộng theo chiều dọc (nâng cấp máy chủ)| Mở rộng theo chiều ngang (thêm nhiều máy chủ) |
    | Tính nhất quán dữ liệu      | Cao (theo nguyên tắc `ACID`)             | Thường chấp nhận nhất quán dần (`Eventually consistent`, `BASE`) |
    | Tính phù hợp                | Dữ liệu có cấu trúc rõ ràng và quan hệ phức tạp | Dữ liệu phi cấu trúc hoặc bán cấu trúc, quy mô lớn |
    | Ví dụ về hệ thống           | `MySQL`, `PostgreSQL`, `Oracle`, `SQL Server` | `MongoDB`, `Cassandra`, `Redis`, `Neo4j` |

### 2. NoSQL Injection là gì?
![image](https://hackmd.io/_uploads/HyZZJxryJe.png)
* `NoSQL injection` là một lỗ hổng cho phép kẻ tấn công can thiệp vào các truy vấn mà ứng dụng thực hiện với cơ sở dữ liệu `NoSQL`. `NoSQL injection` có thể cho phép Attacker:
    * Vượt qua các cơ chế xác thực hoặc bảo vệ.
    * Trích xuất hoặc chỉnh sửa dữ liệu.
    * Gây ra tấn công từ chối dịch vụ (DoS).
    * Thực thi mã trên máy chủ.
    * Cơ sở dữ liệu `NoSQL` lưu trữ và truy xuất dữ liệu theo một định dạng khác so với các bảng quan hệ `SQL truyền thống`. Chúng sử dụng nhiều loại ngôn ngữ truy vấn khác nhau thay vì một tiêu chuẩn chung như `SQL`, và có ít ràng buộc quan hệ hơn.

* Có hai loại NoSQL injection:

    * Injection cú pháp `(Syntax Injection)` - Xảy ra khi ta có thể phá vỡ cú pháp truy vấn `NoSQL`, cho phép chèn payload. Phương pháp này tương tự như `SQL injection`, tuy nhiên bản chất của cuộc tấn công khác nhau đáng kể do cơ sở dữ liệu `NoSQL` sử dụng nhiều ngôn ngữ truy vấn, cú pháp truy vấn và cấu trúc dữ liệu khác nhau.
    * Injection toán tử `(Parameter Injection)` - Xảy ra khi ta có thể sử dụng các toán tử truy vấn của NoSQL để thao tác các truy vấn.

## II. Tìm hiểu cách khai thác NoSQL Injection 
### 1. Injection Cú pháp `(Injection Syntax)`
* Ta có thể phát hiện lỗ hổng `NoSQL injection` bằng cách breakout cú pháp truy vấn. Để làm điều này, hãy kiểm tra từng đầu vào bằng cách gửi các chuỗi `fuzz` và các ký tự đặc biệt để kích hoạt các lỗi Database.
* Ví dụ trong MongoDB:
    * Xem xét một ứng dụng mua sắm hiển thị sản phẩm trong các danh mục khác nhau. Khi người dùng chọn danh mục ví dụ như Bags, Để ý ở thanh url ở Browser sẽ hiện như sau
        ```
        https://insecure-website.com/product/lookup?category=bags
        ```
    * Điều này khiến ứng dụng gửi truy vấn là một chuỗi `JSON` để truy xuất các sản phẩm tương ứng từ bộ sưu tập sản phẩm trong cơ sở dữ liệu MongoDB:
        `this.category == 'fizzy'`
    * Để kiểm tra xem đầu vào có thể bị tấn công hay không, hãy gửi một chuỗi fuzz trong giá trị của tham số category. Một chuỗi ví dụ cho MongoDB là:
        ```
        '"`{
        ;$Foo}
        $Foo \xYZ
        ```
    * Khi đó url sẽ có dạng như sau:
        ```
        https://insecure-website.com/product/lookup?category='%22%60%7b%0d%0a%3b%24Foo%7d%0d%0a%24Foo%20%5cxYZ%00
        ```
    * Nếu ta nhận được bắt kỳ lỗi nào đến từ server thông báo về một lỗi cú pháp từ `Database` thì khả năng cao server đó có thể bị lỗ hỏng `NoSQL Injection` 
* Để xác định các ký tự được ứng dụng hiểu là cú pháp, bạn có thể chèn từng ký tự riêng lẻ. Ví dụ, có thể gửi `'               '`, kết quả là truy vấn MongoDB sau:
    `this.category == '''`
* Nếu điều này gây ra thay đổi từ phản hồi ban đầu, điều này có thể chỉ ra rằng ký tự `'` đã phá vỡ cú pháp truy vấn và gây ra lỗi cú pháp. Bạn có thể xác nhận điều này bằng cách gửi một chuỗi truy vấn hợp lệ, ví dụ bằng cách thoát dấu nháy đơn:
    `this.category == '\''`
* Nếu điều này không gây ra lỗi cú pháp, điều này có thể nghĩa là ứng dụng dễ bị tấn công injection.

* Sau khi phát hiện lỗ hổng, bước tiếp theo là xác định xem bạn có thể ảnh hưởng đến các điều kiện boolean bằng cú pháp `NoSQL` hay không.
* Để kiểm tra điều này, hãy gửi hai yêu cầu, một với điều kiện sai và một với điều kiện đúng. Ví dụ, có thể sử dụng các câu lệnh điều kiện `&& 0 && và && 1 &&` như sau: 
    ```
    https://insecure-website.com/product/lookup?category=bags'+%26%26+0+%26%26+'x
    https://insecure-website.com/product/lookup?category=bags'+%26%26+1+%26%26+'x
    ```
* Nếu server hồi khác nhau, điều này cho thấy điều kiện sai ảnh hưởng đến logic truy vấn, nhưng điều kiện đúng thì không. Điều này chỉ ra rằng payload chèn vào đã ảnh hưởng đến truy vấn trên server.

* Ghi đè các điều kiện hiện có
Khi bạn đã xác định được rằng mình có thể ảnh hưởng đến các điều kiện boolean, bạn có thể thử ghi đè các điều kiện hiện có để khai thác lỗ hổng. Ví dụ, bạn có thể chèn một điều kiện JavaScript luôn đúng, như `'||1||'`:
    `
    https://insecure-website.com/product/lookup?category=bags%27%7c%7c%31%7c%7c%27
    `
* Khi đó truy vấn ở MongoDB sẽ như sau: 
    `this.category == 'bags'||'1'=='1'`
* Vì điều kiện payload luôn đúng, truy vấn đã được sửa đổi sẽ trả về tất cả các mục. Điều này cho phép ta có thể xem tất cả các sản phẩm trong bất kỳ danh mục nào, kể cả những danh mục ẩn hoặc không xác định.

### 2. Injection toán tử `(Parameter Injection)`
* Cơ sở dữ liệu `NoSQL` thường sử dụng toán tử truy vấn, cung cấp cách thức để chỉ định các điều kiện mà dữ liệu phải đáp ứng để được đưa vào kết quả truy vấn. Một số ví dụ về toán tử truy vấn MongoDB bao gồm:
    * `$where`: Khớp với các tài liệu đáp ứng một biểu thức JavaScript.
    * `$ne`: Khớp với tất cả các giá trị không bằng giá trị đã chỉ định.
    * `$in`: Khớp với tất cả các giá trị được chỉ định trong một mảng.
    * `$regex`: Chọn các tài liệu có giá trị khớp với một biểu thức chính quy (regular expression).
* Trong các chuỗi JSON,ta có thể chèn các toán tử truy vấn dưới dạng các đối tượng lồng nhau. Ví dụ, từ `{"username":"trohan"}` có thể trở thành `{"username":{"$ne":"invalid"}}`. Ý nghĩa của payload này dùng để enumerate username trong database và từ có để có thêm thông tin có thể khai thác.
* Nếu cách này không hoạt động,có thể thử các cách sau:
    * Chuyển đổi phương thức yêu cầu từ `GET` sang `POST`.
    * Thay đổi tiêu đề `Content-Type` thành `application/json`.
    * Thêm JSON vào phần thân của thông điệp.
    * Chèn các toán tử truy vấn vào JSON.
* Hãy xem xét một ứng dụng dễ bị tấn công chấp nhận tên người dùng và mật khẩu trong phần thân của yêu cầu POST:
    `{"username":"trohan","password":"trohan0x00"}`
* Kiểm tra từng đầu vào với một loạt các toán tử. Ví dụ, để kiểm tra xem đầu vào username có xử lý toán tử truy vấn hay không, bạn có thể thử chèn như sau: 
    `{"username":{"$ne":"invalid"},"password":"trohan0x00"}`
* Truy vấn sẽ tìm tất cả các `username` khớp với `password` được chỉ định là `trohan0x00` trong database. Nếu ta chưa biết cả `username` và `password` thì có thể sử dụng toán tử `$ne` ở cả hai index.

## III. Thực hành Labs
* Sau khi đã tìm hiểu được cơ chế của lỗ hỏng và cách khai thác thi ta đi đến việc thực hành bằng labs. Các labs ở đây mình sẽ thực hiện trên 4 labs của `Portswigger` về lỗ hỏng này.

### Lab 1: Detecting NoSQL injection
![image](https://hackmd.io/_uploads/rklI6wrkyl.png)
* Với lab đầu tiên thì không có gì gây khó khăn khi đề bài chỉ yêu cầu leak tất cả data về `unreleased products` ở trong database MongoDB ra. Bây giờ ta tiến hành vào lab.
    ![image](https://hackmd.io/_uploads/HkD1RDBkyl.png)
* Vẫn là shop bán đồ dùng quen thuộc của `PortSwigger`=))). Với các danh mực như `Accessories, Clothing, Gifts,...` Ta thử click vào một danh mục bất kỳ để check URL thử.
    `https://0a650053038d2b9681c27fa0007800b4.web-security-academy.net/filter?category=Accessories`
* Đây là URL của server, ta có thể đoán được câu query ở phía MongoDB sẽ là `this.category = "Accessories"` Nếu trường hợp NoSQL Injection xảy ra thì ở phần input nhập vào không được check và ta có thể chèn payload để enum như sau bằng dấu `'`.
    ![image](https://hackmd.io/_uploads/H19y1OHJ1x.png)
* Đúng như dự đoán, server gây ra lỗi cú pháp, tức là việc payload của chúng ta với dạng `this.category = 'Accessories''` đã phá vỡ đi cấu trúc của chuỗi json trong database vì thế lỗi cú pháp đồng nghĩa với việc server đã không hề check input do `user` nhập vào vì thế ta có thể khai thác `NoSQL Injection`.
* Để leak hết dữ liệu từ bảng ở trong SQLI thì ta có thể dùng payload `'OR 1=1--'` còn ở đây NoSQLI ta dùng ở dạng json vì thể để leak dữ liệu ra thì ta có payload `'||1||'` 
