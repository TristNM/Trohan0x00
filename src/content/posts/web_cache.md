---
title: Web Cache Deception
published: 2025-08-01
image: '/src/content/posts/assets/web_cache.png'
tags: [Pentest, Portswigger]
category: 'Learning'
draft: false 
---

# Learning about: Web Cache Deception
![image](https://hackmd.io/_uploads/rys73Tywxg.png)
## I. Web Cache là gì? :spider_web: 
`Web Cache Deception (WCD)` là một kiểu tấn công mạng khai thác lỗ hổng trong cách các hệ thống cache web xử lý và lưu trữ dữ liệu. Mục tiêu chính của `WCD` là đánh lừa bộ nhớ đệm (cache) lưu trữ các thông tin nhạy cảm của người dùng (ví dụ: thông tin cá nhân, phiên làm việc, token CSRF,...) dưới dạng nội dung tĩnh, sau đó cho phép kẻ tấn công truy cập vào các dữ liệu này.
Trước khi đi sâu vào `WCD`, cần hiểu `Web Cache` là gì. `Web Cache` là một hệ thống lưu trữ tạm thời các bản sao của các tài nguyên web (như hình ảnh, tệp CSS, JavaScript, trang HTML) để giảm tải cho máy chủ gốc và tăng tốc độ tải trang cho người dùng. Khi một người dùng yêu cầu một tài nguyên đã được lưu trong cache, máy chủ cache sẽ trả về ngay lập tức mà không cần phải gửi yêu cầu đến máy chủ gốc, giúp tiết kiệm băng thông và giảm độ trễ.
## II. Nguyên nhân gây ra lỗ hỏng
Nguyên nhân gây ra lỗ hỏng bắt nguồn từ cách mà `Web cache` xử lý và tiếp nhận các thông tin và việc cấu hình sai trong việc xử lý các `URL` và trả về thông tin của máy chủ gốc. Nói sau hơn thì ta có tình huống như sau
### 1. Kẻ tấn công lừa bạn truy cập một liên kết đặc biệt:
* Nạn nhân thường truy cập trang hồ sơ cá nhân của mình trên một trang web nào đó, ví dụ: `https://nganhang.com/thong-tin-ca-nhan`. Trang này chứa thông tin mật như số dư tài khoản, lịch sử giao dịch. Đây là một trang động, nghĩa là nội dung của nó sẽ thay đổi tùy theo người dùng.
* Kẻ tấn công sẽ tạo ra một liên kết trông rất "ngây thơ" nhưng lại được thiết kế để đánh lừa bộ nhớ cache, ví dụ: https://nganhang.com/thong-tin-ca-nhan/anh-cua-toi.jpg.
* Kẻ tấn công sẽ gửi liên kết này cho nạn nhân qua email lừa đảo (phishing), tin nhắn, hoặc chèn vào một trang web nào đó để bạn nhấp vào.
### 2. Trình duyệt của nạn nhân gửi yêu cầu `nhập nhằng` và máy chủ trả về thông tin của nạn nhân:
* Khi nạn nhân nhấp vào `https://nganhang.com/thong-tin-ca-nhan/anh-cua-toi.jpg`, trình duyệt sẽ gửi yêu cầu đến máy chủ ngân hàng.
* Máy chủ ngân hàng thấy `thong-tin-ca-nhan` và hiểu rằng muốn xem trang thông tin cá nhân. Nó sẽ tạo ra trang thông tin cá nhân của nạn nhân (bao gồm cả số dư, lịch sử giao dịch) và gửi về cho trình duyệt của bạn.
### 3. Bộ nhớ cache "hiểu nhầm" và lưu thông tin nhạy cảm của bạn vào kho:
* Trên đường dữ liệu từ máy chủ về trình duyệt của nạn nhân, nó sẽ đi qua cái kho chứa tạm thời (bộ nhớ cache) của trang web.
* Khi bộ nhớ cache nhìn thấy yêu cầu `https://nganhang.com/thong-tin-ca-nhan/anh-cua-toi.jpg`, nó thấy có đuôi `.jpg`.
* Đây là điểm mấu chốt: Bộ nhớ cache bị đánh lừa rằng toàn bộ nội dung mà máy chủ vừa trả về (tức là trang thông tin cá nhân nhạy cảm của nạn nhân) là một "bức ảnh" có tên `anh-cua-toi.jpg` và đáng lẽ phải được lưu lại để tải nhanh hơn lần sau.
* Thế là, bộ nhớ cache lưu toàn bộ thông tin tài khoản ngân hàng của nạn nhân dưới tên `anh-cua-toi.jpg`.
### 4.Kẻ tấn công dễ dàng lấy được thông tin của bạn:
* Sau đó, kẻ tấn công chỉ cần truy cập lại chính xác liên kết đó: `https://nganhang.com/thong-tin-ca-nhan/anh-cua-toi.jpg`.
* Lúc này, thay vì phải đi hỏi máy chủ ngân hàng (mà máy chủ sẽ yêu cầu đăng nhập), bộ nhớ cache sẽ thấy "À, tôi có sẵn cái này trong kho rồi!" và trả ngay cái "bức ảnh" chứa thông tin ngân hàng của nạn nhân cho kẻ tấn công mà không hề biết đó là dữ liệu nhạy cảm.
* Vậy là kẻ tấn công đã xem được thông tin cá nhân của bạn mà không cần mật khẩu.


Từ đây ta có thể biết được hơn về nguồn gốc và nguyên nhân gây ra lỗ hỏng này

## III. Cache Keys
Cache Key là một chuỗi định danh duy nhất mà bộ nhớ `cache` sử dụng để xác định và lưu trữ một phản hồi web. Khi một `request` từ trình duyệt đến máy chủ, bộ nhớ `cache` sẽ tạo ra một `Cache Key` dựa trên một số yếu tố của yêu cầu đó. Nếu một yêu cầu tiếp theo khớp với Cache Key đã có trong bộ nhớ cache, thì cache sẽ trả về nội dung đã lưu trữ thay vì gửi yêu cầu đến máy chủ gốc.
Hãy tưởng tượng Cache Key giống như tên của một file trong một kho lưu trữ. Để tìm được đúng file, bạn cần biết đúng tên của nó.
Các yếu tố thường được sử dụng để tạo Cache Key:
* `URL`: Đây là yếu tố quan trọng nhất. Thông thường, toàn bộ hoặc một phần của URL sẽ được dùng làm khóa.
* `Header của HTTP Request`: Một số header cụ thể như Host (tên miền), User-Agent (loại trình duyệt), Accept-Encoding (kiểu mã hóa được hỗ trợ) có thể được thêm vào Cache Key để đảm bảo nội dung được trả về phù hợp với từng loại yêu cầu.
* `Tham số truy vấn (Query Parameters)`: Các tham số sau dấu ? trong URL (ví dụ: ?id=123&sort=asc) cũng thường được đưa vào Cache Key, vì chúng có thể làm thay đổi nội dung của trang.
* `Cookies`: Đôi khi, một số cookie nhất định cũng có thể ảnh hưởng đến Cache Key, đặc biệt nếu nội dung thay đổi dựa trên trạng thái đăng nhập của người dùng.
Trong Web Cache Deception: Kẻ tấn công lợi dụng việc `Cache Key` được tạo ra dựa trên `URL` bao gồm cả phần mở rộng file giả mạo (/anh-cua-toi.jpg). Bộ nhớ cache tạo `Cache Key` cho `https://nganhang.com/thong-tin-ca-nhan/anh-cua-toi.jpg`. Khi nạn nhân truy cập, nội dung nhạy cảm được trả về và gắn với `Cache Key` này. Khi kẻ tấn công truy cập cùng `URL`, `Cache Key` khớp và nội dung nhạy cảm được lấy ra.

## IV. Phát hiện các phản hồi đã được Cache
Trong quá trình kiểm thử, việc nhận biết liệu một phản hồi có được phục vụ từ bộ nhớ `cache` hay từ máy chủ gốc là cực kỳ quan trọng.
Kiểm tra Header phản hồi:
* `X-Cache header`: Đây là một trong những header hữu ích nhất.
    * `X-Cache: hit`: Phản hồi được lấy từ bộ nhớ cache. Đây là dấu hiệu bạn đang tìm kiếm sau khi một yêu cầu đã được cache.
    * `X-Cache: miss`: Phản hồi được lấy từ máy chủ gốc (chưa có trong cache). Trong hầu hết trường hợp, phản hồi này sau đó sẽ được cache. Bạn cần gửi lại yêu cầu một lần nữa để xem nó có chuyển thành hit không.
    * `X-Cache: dynamic`: Nội dung được tạo động từ máy chủ gốc và thường không phù hợp để cache.
    * `X-Cache: refresh`: Nội dung đã được cache nhưng cần được làm mới hoặc xác thực lại với máy chủ gốc.
* `Cache-Control header`: Header này chứa các chỉ thị cho biết cách mà nội dung nên được cache.
    * Ví dụ: `Cache-Control: public, max-age=3600` cho thấy tài nguyên này có thể được cache công khai trong 3600 giây (1 giờ).
    * Lưu ý: Header này chỉ gợi ý khả năng `cache`. Đôi khi bộ nhớ cache có thể bỏ qua hoặc ghi đè lên các chỉ thị này.
* Thời gian phản hồi (Response time):
    * Nếu ta gửi cùng một yêu cầu hai lần liên tiếp và thấy thời gian phản hồi của lần thứ hai nhanh hơn đáng kể so với lần đầu, đây là một dấu hiệu mạnh mẽ cho thấy phản hồi lần thứ hai được phục vụ từ bộ nhớ cache (vì cache nhanh hơn truy vấn máy chủ gốc).

## V. Exploitations
### 1. Exploiting path mapping discrepancies
Phần này mô tả cách kẻ tấn công sẽ thực hiện các bước để tìm ra lỗi `Web Cache Deception`. Nó bao gồm hai giai đoạn chính: kiểm tra máy chủ gốc (origin server) và kiểm tra bộ nhớ cache.
#### Tìm "Điểm Mù" của Máy Chủ Gốc:
Ta cần xác định một địa chỉ web trên trang đích trả về dữ liệu nhạy cảm (ví dụ: thông tin cá nhân, đơn hàng) khi bạn đã đăng nhập.
Sau đó, thử thêm một đoạn chữ bất kỳ (ví dụ: /foo) vào cuối URL đó  `nganhang.com/thong-tin-ca-nhan/foo`
Nếu máy chủ web vẫn trả về cùng một dữ liệu nhạy cảm mà không báo lỗi, điều này cho thấy máy chủ đang bỏ qua đoạn `/foo` đó – đây là một "điểm mù" mà ta có thể lợi dụng.

#### Đánh Lừa Bộ Nhớ Cache:
Với `URL` có đoạn chữ thừa ở trên (ví dụ: nganhang.com/thong-tin-ca-nhan/foo), ta tiếp tục thêm một đuôi file tĩnh giả mạo vào cuối (ví dụ: .css, .jpg).
Kết quả là một URL như: `nganhang.com/thong-tin-ca-nhan/foo.css`.
Ta sẽ kiểm tra bằng cách truy cập `URL` này hai lần. Nếu lần thứ hai nhận được phản hồi nhanh hơn đáng kể (hoặc thấy các header như `X-Cache: hit`), điều đó có nghĩa là bộ nhớ `cache` đã bị đánh lừa và đã lưu trữ phản hồi.
Điều này xác nhận rằng:
* Bộ nhớ cache đã đọc toàn bộ URL (bao gồm /foo.css).
* Bộ nhớ cache có quy tắc tự động lưu trữ các file có đuôi .css.
#### Thu Thập Thông Tin Nhạy Cảm:
* Khi đã xác nhận được cả hai điều trên (máy chủ bỏ qua phần thừa, cache bị lừa bởi đuôi tĩnh), ta có thể tạo một liên kết độc hại tương tự và lừa nạn nhân nhấp vào.
Khi nạn nhân nhấp vào, thông tin nhạy cảm của họ sẽ được máy chủ gốc trả về và bị bộ nhớ cache của trang web lưu trữ dưới tên URL độc hại đó.
Sau đó, kẻ tấn công chỉ cần truy cập lại chính xác URL đó. Bộ nhớ `cache` sẽ "nghĩ" rằng đó là một yêu cầu cho tài nguyên đã lưu và trả lại dữ liệu nhạy cảm của nạn nhân cho kẻ tấn công mà không cần xác thực

#### Lab: Exploiting path mapping for web cache deception
Giờ đến với phần thực hành lab
![image](https://hackmd.io/_uploads/H1qBr--vex.png)
Dựa trên tiêu đề, ta sẽ cần lấy được `API Key` của nạn nhân và ta có một credential riêng là `wiener:peter`. Khi login vào, ta có thể account của ta cũng sẽ có một `API Key`
![image](https://hackmd.io/_uploads/rJIYSZZvge.png)
Có lẽ `API Key` này được gen cho mỗi account, và có thể dùng nó để làm các việc nhạy cảm ( như truy cập tài khoản,...)
Mình sẽ bắt requests vào `Burp` để tiến hành tìm `WCD`. Như trên ta đã tìm hiểu, để tìm được lỗ hỏng, ta cần thực hiện 2 bước là `Xác định endpoint của Origin Server` và `Đánh lừa bộ nhớ Cache`. Ở đây ta đã có được một endpoint đó là `/my-account` rất đáng để thử.
Đương nhiên đây không phải là một URL chưa file tĩnh, nên ta sẽ thêm một file tĩnh giả để trigger thử 
![image](https://hackmd.io/_uploads/B1rXjW-wlg.png)
Ở response thứ nhất, ta đã thấy được rẳng server sẽ cache req này, tức là ta đã tìm ra được điểm mấu chốt để khai thác
![image](https://hackmd.io/_uploads/Hkp8iZWDeg.png)
Đến response thứ 2, ta đã hoàn toàn xác định rằng việc thêm một file tĩnh vào endpoint đã có thể trigger được lỗ hỏng và làm cho `Web cache` lưu trữ thông tin của ta. Để khai thác ở lab này ta sẽ dùng đến `Exploit server`
```html 
<script> document.location="https://0a6e00b20471fcb1e599dc4500f70099.web-security-academy.net/my-account/wcd.js" </script>
```
Đây là payload để có thể dụ nạn nhân click vào vào sau đó chuyển tiếp đến trang endpoint. Sau đó ta có thể dễ dàng truy cập lại đúng URL này bằng browser của ta vì khi này, thông tin đã được lưu ở `Web Cache` trên server
![image](https://hackmd.io/_uploads/ryT93b-Dex.png)
Như có thể thấy, ta đã hoàn toàn lấy được `API Key` của `Carlos`

### 2. Delimiter discrepancies
Ký tự phân cách `Delimiters` là các ký tự đặc biệt dùng để xác định ranh giới giữa các yếu tố khác nhau trong một `URL`. Ví dụ, `?` thường dùng để tách đường dẫn `URL` khỏi chuỗi truy vấn (query string), hoặc `/` dùng để tách các thư mục.
Mặc dù có các tiêu chuẩn (như URI RFC), nhưng các framework và công nghệ khác nhau vẫn có thể diễn giải và sử dụng các ký tự này một cách không nhất quán. Chính sự không nhất quán này tạo ra lỗ hổng `Web Cache Deception`.
#### Vấn đề gây ra lỗ hỏng
##### Sử dụng ; (dấu chấm phẩy) - Ví dụ Java Spring:
`URL`: `/profile;foo.css`
Máy chủ gốc (ví dụ: Java Spring Framework): Java Spring sử dụng `;` để thêm các biến ma trận (matrix variables) vào `URL`. Do đó, khi nó thấy `;`, nó sẽ coi `/profile` là phần chính và bỏ qua phần `;foo.css`. Nó sẽ trả về thông tin hồ sơ của người dùng (nội dung động).

Bộ nhớ Cache (ví dụ: hầu hết các framework khác): Hầu hết các framework hoặc bộ nhớ `cache` khác không sử dụng `;` làm ký tự phân cách đặc biệt. Do đó, chúng sẽ coi toàn bộ `;foo.css` là một phần của đường dẫn. Nếu bộ nhớ cache có quy tắc lưu trữ các phản hồi kết thúc bằng `.css`, nó sẽ lưu trữ nội dung hồ sơ nhạy cảm dưới khóa là `/profile;foo.css`.
Kết quả :heavy_check_mark:: Thông tin hồ sơ nhạy cảm của nạn nhân được `cache` dưới một `URL` có vẻ là file CSS tĩnh, mà kẻ tấn công có thể truy cập.
     
##### Sử dụng . (dấu chấm) - Ví dụ Ruby on Rails:
`Ruby on Rails (RoR)` sử dụng dấu `.` để chỉ định định dạng phản hồi (ví dụ: .html, .json).
`URL`: `/profile`
`RoR` xử lý và trả về thông tin hồ sơ dưới định dạng HTML mặc định.

`URL`: `/profile.css`
RoR nhận ra `.css` là một phần mở rộng nhưng không có bộ định dạng CSS tích hợp sẵn, nên nó trả về lỗi (hoặc không chấp nhận yêu cầu).

`URL`: `/profile.ico`
`RoR` không nhận ra `.ico` là một định dạng hợp lệ để xử lý. Trong trường hợp này, nó sẽ quay về bộ định dạng `HTML` mặc định và vẫn trả về thông tin hồ sơ của người dùng (nội dung động).

Bộ nhớ `Cache`: Nếu bộ nhớ `cache` có quy tắc lưu trữ các phản hồi kết thúc bằng `.ico`, nó sẽ lưu trữ thông tin hồ sơ nhạy cảm đó dưới khóa là `/profile.ico`.
Kết quả :heavy_check_mark: Dữ liệu hồ sơ nhạy cảm bị cache dưới dạng file .ico.

#### Sử dụng ký tự mã hóa (Encoded characters) - Ví dụ %00 (null byte):

`URL`: `/profile%00foo.js`
Máy chủ gốc (ví dụ: OpenLiteSpeed Server): Một số máy chủ như `OpenLiteSpeed` sử dụng ký tự null byte đã mã hóa (%00) làm ký tự phân cách. Nó sẽ hiểu đường dẫn là /profile và bỏ qua phần %00foo.js. Nó sẽ trả về thông tin hồ sơ (nội dung động).

Bộ nhớ `Cache` (ví dụ: Akamai hoặc Fastly CDN): Một số CDN như Akamai hoặc Fastly có thể diễn giải `%00` và mọi thứ sau nó là một phần của đường dẫn bình thường.
Kết quả: Nếu bộ nhớ cache có quy tắc `cache` `.js`, nội dung hồ sơ nhạy cảm sẽ bị cache dưới `/profile%00foo.js`.

> Phương pháp "delimiter discrepancies" là một dạng nâng cao của `Web Cache Deception`. Thay vì chỉ thêm một đuôi file tĩnh vào cuối `URL`, kẻ tấn công tìm ra một ký tự "phân cách" đặc biệt mà máy chủ gốc hiểu theo một cách (bỏ qua phần sau đó) nhưng bộ nhớ cache lại hiểu theo một cách khác (coi đó là một phần của đường dẫn bình thường). Kẻ tấn công lợi dụng sự khác biệt này để chèn thêm đuôi file tĩnh giả mạo, lừa bộ nhớ cache lưu trữ thông tin nhạy cảm.

Đến với việc giải quyết lab, ta cũng sẽ làm tương tự như lab 1 
![image](https://hackmd.io/_uploads/SyOFu5XDle.png)
Ở đây ta sẽ có một list các `delimeter` hữu dụng để khai thác, nhưng mình sẽ tiến hành xem xét sau. Ta sẽ dùng cùng endpoint như lab1 là `/my-account`. Ở đây mình sẽ tiến hành test thử xem nếu dùng như cách khai thác ở lab trước liệu có thành công hay không 
![image](https://hackmd.io/_uploads/SJo0dqQDxx.png)
Như ta thấy, nếu dùng như cách khai thác cũ thì không thành công, do ở đây có lẽ phía server đã không còn cấu hình lỏng lẻo như trước. 
Nhưng nếu ta thêm một `delimeter` thì sao? Để tìm được `delimeter` phù hợp, mình sẽ dùng `Intruder` của Burp để khai thác và tận dụng list được cho
![image](https://hackmd.io/_uploads/Hk4Vt57vex.png)
Ta sẽ được request qua `Intruder` và sau dó thêm checkpoint vào. Giờ là paste toàn bộ list vào 
![image](https://hackmd.io/_uploads/HkD0FcXvel.png)
Và ta nên nhớ cần `deselect` option `URL-Encode` để tránh sai lệch kết quả.
![image](https://hackmd.io/_uploads/ByVW95QDxl.png)
Sau khi brute, ta nhận được hai kết quả là `;` và `?` là đưa về code `200`, tức là phía server đã có thể nhận dạng các `delimeter` này và bỏ qua nó chính là `a.js`. Tiếp theo đó ta cần kiểm tra response của cả 2 để xem `X-Cache` có xuất hiện hay không. 
![image](https://hackmd.io/_uploads/SJwPc5QPlg.png)
Ta thấy rằng ở phía `;` thì có `X-Cache` được bật là `miss` do đây là request đầu tiên. Nhưng ở phía `?` thì mình không thấy tag `X-Cache`. Do đó để có thể khai thác thành công ta cần dùng `;`. 
Ta sẽ gen một request đơn giản và gửi về `Exploti Server` sau đó check lại bằng cách truy cập lại `URL` đó 
![image](https://hackmd.io/_uploads/SyxXls7Del.png)


### 3. Exploiting Static Directory Cache Rules)
Các máy chủ web thường tổ chức các tài nguyên tĩnh (hình ảnh, CSS, JS) vào các thư mục cụ thể như `/static/, /assets/, /scripts/, /images/`. Bộ nhớ `cache` cũng thường có quy tắc để tự động `cache` mọi thứ nằm trong các thư mục này dựa trên tiền tố đường dẫn `URL`.

Nếu kẻ tấn công có thể tạo một `URL` khiến bộ nhớ `cache` tin rằng nó đang truy cập một tài nguyên trong thư mục tĩnh đó, nhưng máy chủ gốc lại hiểu đó là một yêu cầu cho nội dung động nhạy cảm, thì lỗ hổng sẽ xảy ra.
Nguyên lý: Kẻ tấn công sẽ tạo một `URL` có tiền tố là một thư mục tĩnh (ví dụ: /static/), sau đó sử dụng `../` để thoát khỏi thư mục tĩnh đó trong mắt máy chủ gốc và truy cập vào một tài nguyên nhạy cảm.
#### Normalization
Ta cần biết thêm một khái niệm đó là `Normalization` hay còn gọi là `chuẩn hóa`. Là quá trình chuyển đổi các cách biểu diễn khác nhau của một đường dẫn `URL` thành một định dạng chuẩn. Quá trình này có thể bao gồm:
* Giải mã các ký tự đã mã hóa (ví dụ: %2F thành /).
* Giải quyết các `dot-segments` như `../` hoặc `./` .

Vậy cách để phát hiện `chuẩn hoa` trong một máy chủ sẽ như thế nào?
* Gửi một `req` đến một tài nguyên không `cache` được (ví dụ: dùng phương thức POST, hoặc một trang không bao giờ bị cache như /profile).
* Thêm một chuỗi `Path Traversal` và một thư mục tùy ý vào đầu đường dẫn.
    * Ví dụ: Thay đổi `/profile` thành `/aaa/..%2fprofile`.
* Phân tích kết quả:
    * Nếu phản hồi giống hệt phản hồi gốc của `/profile` và trả về thông tin hồ sơ, điều đó có nghĩa là máy chủ gốc đã giải mã %2f thành / và đã giải quyết `../` để chuẩn hóa đường dẫn thành `/profile`. Đây là dấu hiệu của lỗi chuẩn hóa.
    * Nếu phản hồi không khớp (ví dụ: trả về lỗi 404), điều đó có nghĩa là máy chủ gốc không giải mã hoặc không giải quyết dot-segment, và nó hiểu đường dẫn là `/aaa/..%2fprofile`.

> Trong ví dụ trên, mỗi `dot-segment` trong chuỗi `Path Traversal` cần được mã hóa (ví dụ: `..%2f` thay vì `../`). Tại sao? Vì nếu không mã hóa, trình duyệt của nạn nhân sẽ tự động giải quyết `../` trước khi gửi yêu cầu đến bộ nhớ `cache`, khiến cuộc tấn công không thành công. Do đó, để khai thác lỗi chuẩn hóa, yêu cầu là HOẶC bộ nhớ cache HOẶC máy chủ gốc phải giải mã các ký tự trong chuỗi `Path Traversal` (như %2f) và cũng phải giải quyết các dot-segment.

Giờ ta sẽ đến với phần lab thực hành. Ta cũng sẽ được cấp credential như lab trước và việc cần làm là lấy được `API Key` của user `Carlos`
![image](https://hackmd.io/_uploads/rJoobB8Dxe.png)
Với cách khai thác, ta cần phải tìm một directory chứa các file tĩnh mà `Web cache` thường xuyên `cache` lại để tiến hành khai thác. Sau một lúc tìm kiếm thì mình tìm được `/resources` dùng để lưu trữ các file `js` dùng để tracing cho các blogs. Giờ bước đầu tiên ta sẽ cần test thử xem liệu `Normalization discrepancies` có tồn tại hay không.
![image](https://hackmd.io/_uploads/Hy6NtHIPgl.png)
Ở đây sau khi mình test với URL `/resources/..%2fmy-account` thì ta nhận được `res 302` tức là chuyển hướng ta đến trang `/my-account` tức là ở đây `Web server` đã thực hiện `Normalize` kí tự `%2f` thành `/` và thực hiện lệnh lùi thư mục và đưa ta về `my-account` 
![image](https://hackmd.io/_uploads/rk72tr8Dlg.png)
Và thứ ta cần chú ý đến chính là `X-Cache: Miss`, tức là `Web Cache` có hoạt động trong `res` này!!! Do đó ta có thể suy ra rằng tuy `Web server` thực hiện `Normalize` kí tự `%2f` nhưng ở phía `Web cache` thì lại không xử lý kí tự này, nên xem nó là một phần của file ở sau của thư mục chứa các file tĩnh và được `cache` là `resources`, và trạng thái `miss` cho biết đây là lần đầu `request` này được `cache`.
Và ở đây kẻ tấn công có thể lợi dụng điều này để khai thác, và truy cập nội dung với `Cache key` là `URL` hắn đã lừa nạn nhân click vào

Giờ bước tiếp theo là gen payload và gửi đến nạn nhân thôi :smiley: 

### 4. Exploiting normalization by the cache server
Đây là một biến thể ngược lại với biến thể trên do Máy chủ (Origin server) sẽ không thực hiện việc `chuẩn hóa` mà trong khi đó thì `Cache server` lại thực hiện việc đó
* `Bộ nhớ Cache (Cache Server)`: Lại là bên chuẩn hóa đường dẫn URL một cách đầy đủ hơn (tức là nó có khả năng giải mã các ký tự như `%2f` thành `/` và xử lý các dot-segments như ../).
* `Máy chủ Gốc (Origin Server)`: Lại không chuẩn hóa đường dẫn theo cách đó, hoặc xử lý nó khác đi (có thể không giải mã hoặc không giải quyết ../).

Về cơ bản, ở biến thể này ta sẽ làm ngược lại đôi chút so với trước cụ thể 
```URL
/<dynamic-path>%2f%2e%2e%2f<static-directory-prefix> 
```
* `<dynamic-path>`: Đây là phần đường dẫn đến một tài nguyên động, nhạy cảm trên máy chủ gốc. Ví dụ như ở các lab trước là phần `/my-account`
* `%2f%2e%2e%2f`: Đây là chuỗi ký tự đã được mã hóa hoàn toàn của ../.
* Ta phải mã hóa tất cả các ký tự trong chuỗi `Path Traversal` này. Lý do là để đảm bảo trình duyệt của nạn nhân không "thông minh" tự động giải quyết `../` trước khi gửi yêu cầu. Nếu trình duyệt tự giải quyết, nó sẽ làm hỏng payload. Khi chuỗi này được mã hóa, trình duyệt sẽ coi nó là một chuỗi ký tự bình thường và gửi nguyên vẹn.
* `<static-directory-prefix>`: Đây là tiền tố của một thư mục mà bộ nhớ cache có quy tắc để tự động cache các tài nguyên trong đó. Ví dụ: `/static, /assets, /images`. Bộ nhớ `cache` sẽ dùng phần này để quyết định có cache hay không.

> Nhưng đối với payload này thì sẽ không thể khai thác được. Vì khi `Cache server` có thể phân giải và truy xuất đết `/static` thì phía `Origin server` lại xem payload là một `URL` bình thường và sẽ truy xuất đến nó mà không chuẩn hóa. Chính vì vậy nó dẫn đến một `URL` không tồn tại và thứ được `cache` lại sẽ là một trang `404` :smile: 

Để giải quyết vất đề trên, ta sẽ có một cách khai thác khác, là phải tìm được một `delimeter` mà `Ỏrigin server` có thể xử lý như biến thể thứ 2, từ đó nó sẽ bỏ qua toàn bộ chuỗi `Path Traversal` ở sau và tiến hành xử lý một phần `<dynamic-path>` của ta

Giờ tiến hành làm lab, mình sẽ dùng list từ lab trước để scan bằng `Intruder` để tìm `delimeter` phù hợp
![image](https://hackmd.io/_uploads/Bk-6hadPxx.png)
Ta thấy được hai `delimeter` có thể tận dụng là `?` và `#`. OK giờ sẽ đến với việc khai thác. Như thường lệ thì endpoint của ta là `/my-account`, và giờ ta sẽ dùng các `dots-segment` để di chuyển ra phía `resources`
![image](https://hackmd.io/_uploads/BySLa6_wxx.png)
Như ta đẫ thấy là hoàn toàn thành công với payload `/my-account%3f%2f%2e%2e%2fresources`. Ở đây không những cần `URL encode` các `dots-segments` mà còn là kí tự `?` tránh trường hợp bị xử lý ở phía browser như đã nói trước đó
Giờ chỉ cần gửi đến `Exploit server` cho nạn nhân là thành công
### 5. Exploiting file name cache rules
Các máy chủ web thường chứa một số tệp tin phổ biến mà nội dung của chúng ít khi thay đổi, ví dụ như `robots.txt, index.html (trang chủ mặc định), và favicon.ico (biểu tượng trang web)`. Vì những tệp này thường xuyên được yêu cầu và ít khi cập nhật, bộ nhớ `cache (cache server)` thường có các quy tắc đặc biệt để tự động `cache` chúng. Những quy tắc này thường dựa trên việc khớp chính xác tên file.
Cũng giống như việc khai thác các biến thể trước, mình sẽ không nói quá sâu ở đấy chỉ dựa vào các file tĩnh có sẵn và được `cache` để khai thác. Cho nên ta sẽ tiến hành với lab expert luôn
![image](https://hackmd.io/_uploads/BkiN3kFwgg.png)
Lab này yêu cầu ta có thể thay đổi được email của tài khoản `administrator`. Với một lỗ hỏng `WCD` thông thường ta sẽ không thể can thiệp được đối với việc sửa đối thông tin nhưng ở đây ta sẽ tận dụng thêm kiến thức về `CSRF` đã học để khai thác
Bước đầu tiên mình sẽ tìm `delimiter` phù hợp để khai thác, vì ta đã có endpoint như các labs trước là `/my-account` 
![image](https://hackmd.io/_uploads/BJAA31YPgg.png)
Ở đây mình sẽ dùng `;` để khai thác. Giờ ta sẽ thử thêm `dot-segments` vào thử xem khả năng `chuẩn hóa` của `Origin server` và `Cache server` ra sao 
![image](https://hackmd.io/_uploads/S1itAJYveg.png)
Ở đây ta thấy `res` trả về `404` tức là `Origin server` không hề `chuẩn hóa` được `%2f` ta inject vào do đó dẫn đến `URL` sai. Tiếp theo như ta đã biết sẽ có các file tĩnh mặc định như `robots.txt` được `cache` giờ mình sẽ đổi payload đến `robots.txt` 
![image](https://hackmd.io/_uploads/S1POJlFwgx.png)
Header `X-Cache` đã xuất hiện, tức là ở phía `Cache server` đã `chuẩn hoáo` encode này và nhận thấy rằng có `robots.txt` nên đã quyết định `cache lại`. Giờ ta đã có các thông tin cần thiết để khai thác `WCD`. Nhưng ở đây ta cần phải thay đổi được email của `administrator`. 
Ta sẽ thử thay đổi email của mình 
![image](https://hackmd.io/_uploads/HyFYlgKPlg.png)
Thấy rằng mỗi `req` gửi sẽ kèm theo một `CRSF Token` việc cần làm là ta sẽ lấy được `CSRF Token` của `administrator`, vì vậy ta đã có đủ thứ cần để khai thác, giờ ta sẽ gen payload gửi cho nạn nhân
```URL 
/my-account;%2f%2e%2e%2frobots.txt?wcd=1
```
`%3b` sẽ là `URL encoded` của kí tự `;`. Mọi giải thích ở các lab trước mình đã nêu rõ nên giờ sẽ không cần nói thêm nữa 
![image](https://hackmd.io/_uploads/rJBB4eYwee.png)
Ta đã thành công `cache` được `/my-account` của `administrator`, giờ check `res` và tìm `csrf token` 
![image](https://hackmd.io/_uploads/B1uPVetPgg.png)
Giờ ta sẽ dùng `CSRF PoC` để có thể dễ dàng tạo malicious server và dụ nạn nhân để đổi email 
