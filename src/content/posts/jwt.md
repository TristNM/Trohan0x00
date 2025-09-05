---
title: JWT Attacks
published: 2024-12-22
tags: [Portswigger, Pentest]
category: 'Learning'
draft: false 
description: Learning about JWT Attacks
image: "/assets/post_image/jwt.png"
---

# JWT Attacks
## I. Json Web Token (JWT)
![image](https://hackmd.io/_uploads/SkvQ3glBJe.png)
* `JWT (JSON Web Token)` là một phương pháp phổ biến để xác thực và truyền tải thông tin giữa các hệ thống dưới dạng một chuỗi `JSON` có thể xác minh được. `JWT` chủ yếu được sử dụng để xác thực người dùng trong các ứng dụng web hiện đại.
* Khác với các phiên mã hóa thông thường, thì mọi thông tin ở phía `Server` đều được lữu trữ trong `JWT`.
### 1. Format
* JWT bao gồm 3 phần: `Header, Payload và Signature`. Mỗi chúng được phân tách bằng một dấu chấm, như được hiển thị trong ví dụ sau:
![image](https://hackmd.io/_uploads/HJKvUWlByl.png)=
* Mỗi phần lần lượt sẽ gồm các màu sắc tương ứng. Phần `Header` và `Payload` của `JWT` chỉ là các đối tượng `JSON` được mã hóa `base64url`. `Header` chứa `Meta data` về chính mã thông báo, trong khi `Payload` chứa "xác nhận quyền sở hữu" thực tế về người dùng. Ví dụ: ta có decode phần `Payload` từ `JWT` ở trên để hiển thị các xác nhận quyền sở hữu sau:
![image](https://hackmd.io/_uploads/HkplwZlHyx.png)
* Hầu hết phần `Header` và `Payload` đều có thể được decode và đọc bởi bất kỳ ai, do đó tính bảo mật của `JWT` hoàn toàn dựa vào phần cuối cùng đó chính là `Signature` cũng là phần giúp nâng cao độ bảo mật của mã
* Server tạo `JWT` thường tạo phần `Signature` bằng cách băm `Header` và `Payload`. Trong một số trường hợp, sau khi băm, server sẽ mã hóa hàm băm thêm một lần nữa. Dù bằng cách nào, quá trình này liên quan đến `Secret Signature`. Cơ chế này cung cấp cách để máy chủ xác minh rằng không có dữ liệu nào trong `JWT` bị giả mạo
* Ta biết phần `Signature` sẽ được gen ra dựa vào `Payload` và `Header` trước đó, do đó nếu có sửa đổi 1 trong 2 phần đó thì sẽ dẫn đến việc phần `Signature` sẽ không khớp và nếu ta không biết được `Secret Key` của `Signature` ta sẽ không thể tạo `Signature` chính xác cho từng `Header` hoặc `Payload` nhất định
#### 2. Công dụng của JWT
##### Xác thực người dùng (Authentication)
* `JWT` là một trong những phương pháp phổ biến để xác thực người dùng, đặc biệt trong các ứng dụng web và `API`.
* Cách hoạt động:
    * Sau khi người dùng đăng nhập, server tạo một `JWT` chứa thông tin người dùng (ví dụ: userId, username) và gửi lại cho trình duyệt hoặc ứng dụng.
    * `JWT` được lưu trữ (thường trong localStorage, sessionStorage, hoặc cookie).
Mỗi lần người dùng thực hiện yêu cầu, `JWT` sẽ được gửi cùng (trong header HTTP hoặc cookie) để server xác nhận danh tính.
Ưu điểm:
* Stateless: `Server` không cần lưu trữ phiên làm việc (session) vì mọi thông tin cần thiết đã nằm trong `JWT`.
* Dễ dàng tích hợp với các ứng dụng hiện đại như `SPA (Single-Page Application)`.
##### Phân quyền (Authorization)
* `JWT` không chỉ xác thực danh tính người dùng mà còn giúp phân quyền trong hệ thống.
* Ví dụ:
    * Một JWT có thể chứa thông tin về quyền hạn của người dùng:
```json
{
    "userId": 123,
    "role": "admin",
    "permissions": ["read", "write", "delete"]
}
```
* Dựa vào thông tin này, server quyết định người dùng có được phép truy cập tài nguyên hay không.
##### Trao đổi thông tin an toàn (Secure Data Exchange)
* `JWT` có thể truyền thông tin an toàn giữa các bên.
* Cách dùng:
    * `JWT` được ký (signed) bằng thuật toán như `HMAC` hoặc `RSA`, đảm bảo tính toàn vẹn (integrity).
    * Dữ liệu trong `JWT` không thể bị thay đổi nếu không có khóa bí mật hoặc khóa công khai tương ứng.
* Ví dụ:
    * Gửi yêu cầu xác minh email chứa JWT trong URL.
    * Truyền thông tin tạm thời giữa các dịch vụ mà không cần lưu trữ trạng thái.

## II. Khai thác lỗ hỏng với JWT
### 1. Không xác thực Signature
* Lỗ hỏng này bắt nguồn từ việc server không lưu thông tin về các `JWT` mà nó đã phát hành. Điều này tiện lợi vì không cần duy trì trạng thái (stateless). Tuy nhiên, server chỉ có thể kiểm tra tính hợp lệ của `JWT` bằng cách xác minh chữ ký `(Signature)`.
* Hoặc nếu server không kiểm tra chữ ký hoặc sử dụng cách kiểm tra không đúng, kẻ tấn công có thể:
    * Chỉnh sửa `payload` (dữ liệu trong JWT) để thay đổi thông tin.
    * Giả mạo thông tin nhạy cảm như quyền truy cập.
* Ví dụ:
* Giả sử một JWT chứa payload như sau:
```json
{
    "username": "carlos",
    "isAdmin": false
}
```
* Payload này chứa thông tin về người dùng:
    * `username`: Xác định tên người dùng đang đăng nhập.
    * `isAdmin`: Xác định quyền truy cập (admin hay không).
* Nếu server chỉ dựa vào username để xác định danh tính thì kẻ tấn công có thể sửa `username` thành một tên khác, ví dụ: `"username": "admin"`, để giả mạo làm người khác.
* Nếu server dùng `isAdmin` để phân quyền thì kẻ tấn công có thể sửa `isAdmin` thành `true` để nâng quyền truy cập.
#### Lỗ hổng: Chấp nhận chữ ký tùy ý
* Các thư viện `JWT` thường cung cấp 2 phương thức:
    * `decode()`: Giải mã JWT mà không kiểm tra chữ ký.
    * `verify()`: Giải mã JWT và kiểm tra chữ ký.
* Nhưng đôi khi, nhà phát triển nhầm lẫn và chỉ dùng `decode()`. Điều này dẫn đến việc `JWT` được giải mã, nhưng chữ ký không được xác minh. Kẻ tấn công có thể sửa chữ ký hoặc `payload`, vì server không kiểm tra tính hợp lệ.
#### LAB: JWT authentication bypass via unverified signature
![image](https://hackmd.io/_uploads/By_1PlNSyg.png)
* Ta sẽ tiến hành thực hành lab này dựa trên lỗ hỏng như đã nói
* Đăng nhập vào lab với credentials là `wiener:peter` ta sẽ được cấp 1 `JWT` như 1 cookie.
![image](https://hackmd.io/_uploads/HktmyBNBkg.png)
* Mình sẽ sử dụng một extension của `Burp` là `JWT edit` để có thể thêm, xóa, sửa đối với `JWT` nhưng để có cái nhìn rõ hơn mình sẽ ưu tiên dùng [jwt.io](https://jwt.io) để mọi người có thể nhìn rõ `JWT`
![image](https://hackmd.io/_uploads/H1u91rNBkl.png)
* Như ta thấy 3 phần với 3 màu khác nhau lần lượt sẽ là `Header, Payload, Signature`. Như đã nói ở trên về lỗ hỏng, thì dev đã không thực hiện việc xác minh chỉ ký bằng cách dùng hàm `Verify()` thay vào đó thì lại dùng `Decode()` nên ta có thể sửa đổi lại `JWT` này mà k cần lo đến việc sai lệch chữ ký. ở phần `sub` mình sẽ sửa lại thành `"sub" : "administrator"` từ đó ta có một chuỗi `JWT` mới và tiến hành vào `/admin` xemm thử
![image](https://hackmd.io/_uploads/HypElBNr1l.png)
* Như đã thấy, ta đã có thể truy cập vào đường dẫn của `admin` giờ chỉ cần gửi request để delete carlos cùng với `JWT`đã có
![image](https://hackmd.io/_uploads/HJudxBEr1l.png)
### 2. Không lọc các JWT thiếu chữ ký
* Đúng như tên gọi, ở đây server đã bỏ quên việc kiểm tra và loại bỏ các `JWT` bị thiếu chữ ký cụ thể:
* Phần header của JWT thường chứa tham số `alg (algorithm)` để chỉ định thuật toán được dùng ký token. Ví dụ:
```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```
* `HS256`: Thuật toán ký dùng `HMAC` với `SHA-256`.
* `typ`: Loại token (ở đây là JWT).
* Vấn đề: Máy chủ tin tưởng nội dung của header mà không kiểm tra ngay, dẫn đến kẻ tấn công có thể sửa giá trị `alg` để thay đổi cách máy chủ xác minh chữ ký.
* Thông thường, server không chấp nhận token không có chữ ký. Nhưng nếu kẻ tấn công sửa header thành:
```json
{
    "alg": "none",
    "typ": "JWT"
}
```
* `alg`: `none` có nghĩa là token không cần chữ ký.
* Một số server không kiểm tra kỹ và chấp nhận token dù không có chữ ký.
#### LAB: JWT authentication bypass via flawed signature verification
![image](https://hackmd.io/_uploads/rksiZHVSyx.png)
* Ta sẽ dùng extension để sửa `Header` và `Payload` như sau
![image](https://hackmd.io/_uploads/B1187SErJl.png)
* sửa `alg` thành `none` sau đó là chỉnh `sub` ở `Payload` như lab trước sau đó ta sẽ xóa đi phần `Signature` của `JWT` nhưng nên nhớ là sẽ để lại dấu `.` ở cuối để `JWT` không bị lỗi 
![image](https://hackmd.io/_uploads/SJQ67rEByg.png)
* Như vậy ta đã có thể Bypass vào `/admin` thành công
### 3. Brute Force Secret Key
* Khi triển khai các ứng dụng `JWT`, các lập trình viên đôi khi mắc phải những sai lầm như quên thay đổi `Secret key` mặc định `Placeholder`. Họ thậm chí có thể `copy&paste` các đoạn mã mà họ tìm thấy trên mạng, sau đó quên thay đổi `Secret key` được mã hóa cứng được cung cấp. Trong trường hợp này, có thể rất dễ dàng đối với kẻ tấn công để brute force `Secret key` của máy chủ bằng cách sử dụng các wordlist nổi tiếng về việc brute force `JWT` như [jwt.secrets.list](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list)
#### Brute force bằng hash cat
* `Hashcat` là một công cụ mạnh mẽ dùng để `brute-force` các loại `hash`, trong đó có thể sử dụng để tấn công `JWT (JSON Web Token)` với `signing key` yếu hoặc phổ biến. `JWT` thường được bảo mật bằng cách sử dụng một `secret key` để ký kết hợp lệ phần `Header` và `Payload` của token. Nếu `secret key` này yếu hoặc nằm trong một wordlist, ta có thể brute-force để tìm ra nó.
* Các bước thực hiện
    ```shell
    hashcat -a 0 -m 16500 <jwt> <wordlist>
    ```
    * `-a 0`: Chỉ định chế độ attack (mode 0: dictionary attack).
    * `-m 16500`: Mode dành cho JWT.
    * `<jwt>`: JWT bạn muốn brute-force.
    * `<wordlist>`: Đường dẫn đến file wordlist chứa các secret keys.
* Cơ chế hoạt động
    * `Hashcat` kiểm tra từng secret key trong wordlist:
    * Lấy `header` và `payload` từ `JWT`.
    * Dùng `secret key` để tạo lại `signature`.
    * So sánh `signature` vừa tạo với `signature` gốc của `JWT`. Nếu khớp, `hashcat` sẽ xuất ra `secret key`.
#### LAB: JWT authentication bypass via weak signing key
![image](https://hackmd.io/_uploads/BJVsl4Brke.png)
* Với lab này ta sẽ thực hiện việc brute `secret key` và gen lại `Signature`.  
![image](https://hackmd.io/_uploads/SJYqbVrHkl.png)
* Sau khi lấy được `JWT` bây giờ ta sẽ tiến hành brute bằng `Hashcat`. có câu lệnh như sau:
```shell
 hashcat -a 0 -m 16500 eyJraWQiOiJmNmUyOTc2NC01MWNlLTRkMjktOTNhNi1jN2M0ZDQ3YmExNjAiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTczNDg1MjQwOSwic3ViIjoid2llbmVyIn0.dri
B3zqlqwbpqfx4yBKvCxyBZTF2R4WHO-Ow3DVj53s ~/tools/SecLists/Discovery/Web-Content/jwt.secrets.list 
```
![image](https://hackmd.io/_uploads/Byhzz4SSye.png)
* Giờ ta lấy được phần `secret key = secret1`, từ đây mình sẽ dùng `JWT editor` để gen lại phần `Signature` 
![image](https://hackmd.io/_uploads/Hky_GErHJe.png)
* Ta vào `JWT editor` và nhập `secret` vào phần `specify secret` sau đó ở phần `Repeater` ta sẽ có thêm phần `JSON Web Token` click vào đi đến phần `Sign
![image](https://hackmd.io/_uploads/SkApzVBHJe.png)
* Tiếp theo là chọn phần `secret key` vừa gen ở `JWT editor` và ấn ok, giờ ta đã có được phần `Signature` mới và giờ chỉ cần sửa lại `Payload` là được
![image](https://hackmd.io/_uploads/HkjL7NBH1g.png)
### 4. JWT header parameter injection
* Theo đặc tả của **JWS** (JSON Web Signature), chỉ tham số header `alg` là bắt buộc. Tuy nhiên, trong thực tế, header của JWT (còn gọi là **JOSE headers**) thường chứa nhiều tham số khác. Một số tham số đáng quan tâm đối với kẻ tấn công bao gồm:

    - **`jwk` (JSON Web Key)**: Cung cấp một đối tượng JSON nhúng đại diện cho khóa.
    - **`jku` (JSON Web Key Set URL)**: Cung cấp URL để server lấy tập hợp khóa chứa khóa đúng.
    - **`kid` (Key ID)**: Cung cấp ID mà server có thể sử dụng để xác định khóa phù hợp khi có nhiều khóa để chọn.
* Những tham số này, do người dùng kiểm soát, sẽ chỉ ra cho server nhận biết khóa nào cần dùng để xác minh chữ ký. Dưới đây là cách khai thác các tham số này để tạo JWT đã chỉnh sửa và ký bằng khóa tùy ý của bạn thay vì khóa bí mật của server.

---

#### Chèn JWT tự ký qua tham số `jwk`
* Đặc tả `JWS` cho phép sử dụng tham số header `jwk` để nhúng trực tiếp khóa công khai vào token dưới định dạng **JWK**.
* **JWK (JSON Web Key)** là định dạng chuẩn hóa để biểu diễn khóa dưới dạng một đối tượng JSON.
Ví dụ về header JWT sử dụng `jwk`:

```json
{
    "kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",
    "typ": "JWT",
    "alg": "RS256",
    "jwk": {
        "kty": "RSA",
        "e": "AQAB",
        "kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",
        "n": "yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9m"
    }
}
```

##### Khóa công khai và khóa riêng tư
* Khóa công khai (public key) và khóa riêng tư (private key) là hai thành phần quan trọng trong mật mã bất đối xứng.
* Server lý tưởng nên chỉ sử dụng danh sách khóa công khai được cho phép để xác minh `JWT`.
* Tuy nhiên, server bị cấu hình sai có thể chấp nhận bất kỳ khóa nào được nhúng trong tham số jwk.
* Cách khai thác:
    - Tạo JWT đã chỉnh sửa và ký bằng khóa riêng tư RSA của bạn.
    - Nhúng khóa công khai tương ứng vào header jwk.
* Sử dụng Burp Suite:
    - Tải `JWT Editor extension`.
    - Trong tab `JWT Editor Keys`, tạo khóa RSA mới.
    - Gửi yêu cầu chứa `JWT` đến Burp Repeater.
    - Chuyển sang tab JSON Web Token, chỉnh sửa payload token.
    - Nhấn Attack, chọn Embedded JWK, và gửi yêu cầu để kiểm tra phản hồi từ server.
#### Chèn JWT tự ký qua tham số jku
* Thay vì nhúng trực tiếp khóa công khai trong jwk, một số server cho phép sử dụng jku để chỉ định URL chứa JWK Set.
* JWK Set là gì?
JWK Set là một đối tượng JSON chứa mảng các JWK đại diện cho nhiều khóa khác nhau. Ví dụ:
```json
{
    "keys": [
        {
            "kty": "RSA",
            "e": "AQAB",
            "kid": "75d0ef47-af89-47a9-9061-7c02a610d5ab",
            "n": "o-yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9mk6GPM9gNN4Y_qTVX67WhsN3JvaFYw-fhvsWQ"
        },
        ...
    ]
}
```
* Cách khai thác:
    - Lợi dụng server bị cấu hình sai để server tải tập hợp khóa từ `URL` do bạn kiểm soát.
    - Ký `JWT` bằng khóa riêng của bạn và cung cấp URL chứa khóa tương ứng.
#### Chèn JWT tự ký qua tham số kid
* Tham số kid giúp server xác định khóa nào cần dùng khi xác minh chữ ký.
* Cách khai thác:
    - Sử dụng directory traversal để chỉ định đường dẫn đến một file trên hệ thống server, ví dụ:
```json
{
    "kid": "../../path/to/file",
    "typ": "JWT",
    "alg": "HS256",
    "k": "asGsADas3421-dfh9DGN-AFDFDbasfd8-anfjkvc"
}
```
* Trong trường hợp server hỗ trợ thuật toán đối xứng:
* Trỏ kid đến file tĩnh và dễ đoán như /dev/null.
* Ký `JWT` bằng chuỗi rỗng, vì đọc từ /dev/null trả về chuỗi rỗng.
> Lưu ý: JWT Editor không hỗ trợ ký bằng chuỗi rỗng. Tuy nhiên, bạn có thể tận dụng lỗi của extension bằng cách dùng ký tự null byte đã mã hóa Base64.


