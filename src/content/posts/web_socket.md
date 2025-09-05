---
title: Web Socket Vulnerability
published: 2025-07-30
image: '/assets/post_image/ws.png'
tags: [Portswigger, Pentest]
category: 'Learning'
draft: false 
---


# Learning about: Web Socket Vulnerabilties
## I. WebSocket là gì?
![image](https://hackmd.io/_uploads/S1b3YGYIgg.png)
Đầu tiên ta cần có cái nhìn hiểu biết vể `Websocket`. `Websocket` đơn giản là một giao thức truyền tin `full duplex` được khởi tạo qua `HTTP`. 
Khác với `HTTP` thì `Websocket` cho phép cả `client` và `server` gửi dữ liệu cho nhau bất cứ lúc nào khi hoàn thành kết nối, không cần phải đợi nhận `req` và phản hồi `res`. Chính vì những ưu điểm trên so với `HTTP`, `Websocket` mang lại những lợi ích sau:
* Giao tiếp thời gian thực: Cho phép truyền dữ liệu ngay lập tức giữa máy chủ và máy khách mà không cần `polling` liên tục.
* Hiệu quả cao: Giảm thiểu `overhead` do không cần thiết lập lại kết nối cho mỗi lần trao đổi dữ liệu, tiết kiệm băng thông và tài nguyên máy chủ.
* Tính linh hoạt: Có thể truyền nhiều loại dữ liệu khác nhau (text, binary) một cách dễ dàng.
## II. Khác nhau giữa HTTP và WebSocket
Hầu hết giao tiếp giữa web brwoser và trang web đều sử dụng `HTTP`. Với `HTTP`, `client` gửi `req` và `server` trả về `res`. Thông thường, phản hồi xảy ra ngay lập tức và giao dịch hoàn tất. Ngay cả khi kết nối mạng vẫn mở, điều này sẽ được sử dụng cho một giao dịch riêng biệt của một yêu cầu và phản hồi.

| Đặc điểm           | HTTP (Giao thức truyền thống)                                  | WebSocket (Giao thức thời gian thực)                                |
| :----------------- | :-------------------------------------------------------------- | :------------------------------------------------------------------ |
| **Mô hình giao tiếp** | **Một chiều**, **yêu cầu-phản hồi** (Máy chủ chỉ trả lời khi được hỏi) | **Hai chiều**, **liên tục** (Cả hai bên có thể gửi/nhận bất cứ lúc nào) |
| **Kết nối** | **Ngắn hạn** (Mở, gửi, nhận, đóng/giữ ngắn)                    | **Dài hạn** (Duy trì một kết nối sau bắt tay ban đầu)             |
| **Phù hợp cho** | Tải trang web, hình ảnh, video; ứng dụng không cần cập nhật liên tục (đọc báo, blog) | Trò chuyện trực tuyến, game online, cập nhật dữ liệu trực tiếp, thông báo tức thì |
| **Ưu/Nhược điểm** | **Nhược điểm:** Tốn tài nguyên, chậm cho giao tiếp thời gian thực | **Ưu điểm:** Tốc độ cao, độ trễ thấp, hiệu quả cho real-time       |

## III. Khai thác các lỗ hỏng WebSocket
Về nguyên tắc, thực tế, bất kỳ lỗ hổng bảo mật web nào cũng có thể phát sinh liên quan đến WebSockets
### 1. Manipulating WebSocket messages to exploit vulnerabilities
Phần lớn các lỗ hổng dựa trên đầu vào ảnh hưởng đến `WebSockets` có thể được tìm thấy và khai thác bằng cách giả mạo nội dung của các tin nhắn websocket 
VÍ dụ như khi ta gửi một nội dung thông qua websocket, khi ta nhập một tin nhắn websocket ví dụ như `{"message":"Hi"}` thì sau đó tin nhắn đến được server và sẽ được `reflect` lại đối với người dùng khác.
Trong tình huống này, việc không có validate input đầu vào thì kẻ tấn công có thể thực hiện một cuộc tấn công `XSS` bằng cách gửi tin nhắn WebSocket với các payload `XSS` tùy theo ngữ cảnh
Đến với lab thực hành, ở đây ta sẽ có một tính năng là `live chat` dùng để chat trực tiếp với người bán và tính năng này sử dụng `Websocket` 
![image](https://hackmd.io/_uploads/BJfNgNYUgg.png)
Check source thì ta thấy tính năng này được xử lý bởi file `chat.js` và ta sẽ sử dụng kỹ thuật manipulate để khai thác. Ở `Burp` ta sẽ bật intercept lên và sau đó vào phần `Websocket history` có thể xem được các gói tin của ws. Ta thử gửi một payload như một thẻ `HTML` thử 
![image](https://hackmd.io/_uploads/HJd6xVKLlx.png)
Như ta có thể thấy ở `To Server` thì payload này đã bị `HTML Encoded` và khả năng ở phần xử lý dữ liệu hiển thị cho người bán sẽ có thể thực thi được các thẻ `HTML` vậy nên ta có thể trigger XSS bằng cách inject payload `HTML` vào
```html 
<img src=1 onerror='alert(1)'>
```
Ta sẽ tiến hành dùng repeater để sửa lại phần request của `websocket`
![image](https://hackmd.io/_uploads/SJ5vGEYUxg.png)
![image](https://hackmd.io/_uploads/Sk3vMEK8lx.png)

### 2. Manipulating the WebSocket handshake to exploit vulnerabilities
Quá trình bắt tay `WebSocket` là bước đầu tiên để thiết lập kết nối `WebSocket`, diễn ra trên nền `HTTP`. Kẻ tấn công có thể lợi dụng lỗi thiết kế trong bước này để khai thác các lỗ hổng:
Tin tưởng sai lầm vào `HTTP header` (ví dụ: X-Forwarded-For):
* Vấn đề: Ứng dụng tin tưởng các header HTTP dễ bị giả mạo (như X-Forwarded-For) để đưa ra quyết định bảo mật (ví dụ: kiểm soát IP, ghi log).
* Khai thác: Kẻ tấn công giả mạo header này trong yêu cầu bắt tay để vượt qua kiểm soát truy cập hoặc ẩn danh.
Lỗi trong xử lý phiên (Session handling):
* Vấn đề: Ngữ cảnh phiên của WebSocket được xác định bởi phiên của tin nhắn bắt tay.
* Khai thác: Nếu kẻ tấn công chiếm được cookie/token phiên của nạn nhân, họ có thể sử dụng chúng trong yêu cầu bắt tay để thiết lập kết nối `WebSocket` với quyền của nạn nhân (ví dụ: Cross-Site WebSocket Hijacking - CSWSH).
Bề mặt tấn công từ `HTTP` header tùy chỉnh:
* Vấn đề: Ứng dụng dùng các header HTTP tùy chỉnh (không chuẩn) để truyền thông tin nhạy cảm nhưng không xác thực kỹ.
* Khai thác: Kẻ tấn công chèn mã độc (injection) hoặc thay đổi giá trị của các header này để thao túng logic ứng dụng hoặc vượt qua kiểm soát.

Đến với lab này, ta cũng sẽ thử trigger XSS bằng cách cũ đó chính là dùng `<img src=1 onerror='alert(1)'>` 
![image](https://hackmd.io/_uploads/ByBsd8KLgl.png)
Như ta thấy, ta đã bị detected và bị ngắt kết nối khỏi server bằng cách là cho IP của ta vào blacklist
![image](https://hackmd.io/_uploads/H1YTuUY8le.png)
Từ đây ta có thể bypas này bằng cách dùng `X-Forwarded-For`
![image](https://hackmd.io/_uploads/B1ZNY8FLge.png)
Ta đã có thể vào lại server, nhưng điểm khó khăn tiếp theo là việc bypass được filtered XSS vì cho dù ta khong bị detect nhưng mỗi lần inject vào ta đều bị disconnected. Sau một lâu tìm kiếm thì mình có payload 
```
<img src=1 oNeRrOr=alert`1`>
```
Vì căn bản bộ lọc của server chỉ filtered mỗi các tag và attribute bị cho vào blacklist nhưng nó không chặt chẽ, nên ta có thể dễ dàng bằng cách lồng các khoảng trắng hoặc kí tự vào các tag và attributes này

### 3. Cross-site WebSocket Hijacking (CSWSH) là gì?
`Cross-site WebSocket Hijacking (CSWSH)`, hay còn gọi là `Cross-origin WebSocket Hijacking`, là một lỗ hổng `CSRF` nhưng xảy ra trong quá trình `Websocket handshake`.
Hãy hình dung thế này:
* Khi ta truy cập một trang web có sử dụng `WebSocket` (ví dụ như là trang có livechat ở các labs trước), trình duyệt sẽ gửi request `Websocket handshake` để bắt tay và thiết lập kết nối `WebSocket` với máy chủ.
* Máy chủ thường dựa vào cookie `HTTP` để xác định ta là ai. Vấn đề xảy ra khi quá trình bắt tay này chỉ dựa vào `cookie` để xác thực phiên và không có bất kỳ biện pháp bảo vệ `CSRF` nào khác.
* Kẻ tấn công có thể tạo ra một malicious web riêng. Khi ta truy cập trang này, trang malicious web sẽ cố gắng thiết lập một kết nối `WebSocket` chéo trang đến trang web có `WebSocket` mà ta đã nhập trước đó.

