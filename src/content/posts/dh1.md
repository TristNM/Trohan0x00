---
title: DreamHack Web level 1 (1)
published: 2024-10-10
description: Learning about NoSQL Injection
tags: [Pentest]
category: 'Writeup'
draft: false 
---

# Writeup DreamHack Web Level 1 Part 1

## Baby case
![image](https://hackmd.io/_uploads/BJY6EJHgkl.png)
* Source gồm hai route là root và `/shop`. Truy cập vào `/shop` thì ta sẽ bị từ chối truy cập và trả về 403.
* Cách để bypass thì ta sẽ đổi thành `/SHOP` và dùng python để gửi request kèm data
```py
import requests

url = "http://host3.dreamhack.games:15417/"

data = {
    'leg' : 'flag'
}

res = requests.post(url+'SHOP', data=data)
print(res.text)
```
![image](https://hackmd.io/_uploads/BygNSkBeke.png)
> DH{167e709c31556e37207e46db78f34ab1d6919e0789e3441b5740f02143169ecc}

## 2. simple-phparse
```py
<html>
<head>
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
    <title>PHParse</title>
</head>
<body>


    <!-- php code -->
    <?php
     $url = $_SERVER['REQUEST_URI'];
     $host = parse_url($url,PHP_URL_HOST);
     $path = parse_url($url,PHP_URL_PATH);
     $query = parse_url($url,PHP_URL_QUERY);
     echo "<div><h1> host: $host <br> path: $path <br> query: $query<br></h1></div>";

     if(preg_match("/flag.php/i", $path)){
        echo "<div><h1>NO....</h1></div>";
     }
     else echo "<div><h1>Cannot access flag.php: $path </h1></div> ";
    ?> 

<style type="text/css">
        body {
            margin: 1em;
        }
        div {
            margin: 0 5px 0 0;
            padding: 0.1em;
            border: 2px solid silver;
            border-radius: 7px;
        }

</style>
</body>
</html>
```
* Source đơn giản dùng parse để lấy các giá trị như `host, path, query` sau đó detect `/flag.php` từ người dùng. Ban đầu mình khá loay hoay khi k thấy được code từ đề dùng để import file flag.php vào. Sau đó thì thử bypass bằng các cách khác thì lấy được flag bằng payload `/%66lag.php`
![image](https://hackmd.io/_uploads/rJp38JSlke.png)
>  DH{0DB34CA258F33CDC4928EC7EAD7FAD468D2DD255805FF183927A7A15772745FB}

## 3. what is my ip
* Web sẽ truy xuất ip từ máy của user và in ra web 
```py
#!/usr/bin/python3
import os
from subprocess import run, TimeoutExpired
from flask import Flask, request, render_template

app = Flask(__name__)
app.secret_key = os.urandom(64)


@app.route('/')
def flag():
    user_ip = request.access_route[0] if request.access_route else request.remote_addr
    try:
        result = run(
            ["/bin/bash", "-c", f"echo {user_ip}"],
            capture_output=True,
            text=True,
            timeout=3,
        )
        return render_template("ip.html", result=result.stdout)

    except TimeoutExpired:
        return render_template("ip.html", result="Timeout!")


app.run(host='0.0.0.0', port=3000)
```
* Ta thấy được chỉ có 1 route là root và biến `user_ip` dùng để lưu ip của người dùng. Lỗ hỏng ở đây là đoạn `["/bin/bash", "-c", f"echo {user_ip}"]` khi đưa thẳng `user_ip` vào hàm `run()` nhưng không detect hay filter do đó ta có thể lợi dụng để thực hiện `CMD Injection`
* Nhưng vấn đề là làm sao để có thể tương tác được với biến `user_ip` trong khi server lại không yêu cầu bất kỳ input nào. Do đó ta có thể dùng `X-Forwarded-For` để thực hiện chỉ định ip trong request. Payload cụ thể sẽ như sau `IP; cat /flag;`
![image](https://hackmd.io/_uploads/H1ZC2yHeyg.png)
> DH{1acfe9db38697eb71538e97e71882f1ad6deb5cb9d8c3448bd05d3adb805e559}

## 4. BypassIF
![image](https://hackmd.io/_uploads/S1NP6ySx1e.png)
* Yêu cầu ta cần nhập key để lấy flag 
```py
#!/usr/bin/env python3
import subprocess
from flask import Flask, request, render_template, redirect, url_for
import string
import os
import hashlib

app = Flask(__name__)

try:
    FLAG = open("./flag.txt", "r").read()
except:
    FLAG = "[**FLAG**]"

KEY = hashlib.md5(FLAG.encode()).hexdigest()
guest_key = hashlib.md5(b"guest").hexdigest()

# filtering
def filter_cmd(cmd):
    alphabet = list(string.ascii_lowercase)
    alphabet.extend([' '])
    num = '0123456789'
    alphabet.extend(num)
    command_list = ['flag','cat','chmod','head','tail','less','awk','more','grep']

    for c in command_list:
        if c in cmd:
            return True
    for c in cmd:
        if c not in alphabet:
            return True

@app.route('/', methods=['GET', 'POST'])
def index():
    # GET request
    return render_template('index.html')



@app.route('/flag', methods=['POST'])
def flag():
     # POST request
    if request.method == 'POST':
        key = request.form.get('key', '')
        cmd = request.form.get('cmd_input', '')
        if cmd == '' and key == KEY:
            return render_template('flag.html', txt=FLAG)
        elif cmd == '' and key == guest_key:
            return render_template('guest.html', txt=f"guest key: {guest_key}")
        if cmd != '' or key == KEY:
            if not filter_cmd(cmd):
                try:
                    output = subprocess.check_output(['/bin/sh', '-c', cmd], timeout=5)
                    return render_template('flag.html', txt=output.decode('utf-8'))
                except subprocess.TimeoutExpired:
                    return render_template('flag.html', txt=f'Timeout! Your key: {KEY}')
                except subprocess.CalledProcessError:
                    return render_template('flag.html', txt="Error!")
            return render_template('flag.html')
        else:
            return redirect('/')
    else: 
        return render_template('flag.html')


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000, debug=True)
```
* Quan sát source ta thấy được route /flag bao gồm biến `key` và `cmd_input` dùng để check ở bên trên có đoạn `KEY = hashlib.md5(FLAG.encode()).hexdigest()
guest_key = hashlib.md5(b"guest").hexdigest()` từ đây ta biết được guest_ket sẽ là hash md5 của `"guest"` enc ra sẽ được chuỗi `23e23e08be4bfff509e16a63784ee015`. Tiến hành nhập thử key vào nhưng kết quả vẫn sẽ return về `index.html`
* Dùng Burp ta bắt request và thấy được phần request `Post` chỉ gửi đi mỗi biến `key` và k có `cmd_input`
![image](https://hackmd.io/_uploads/SyIaRkHgkl.png)
* Thêm thử `cmd_input` vào request thực hiện thử `cmd_injection` do đoạn `output = subprocess.check_output(['/bin/sh', '-c', cmd], timeout=5)` không hề có filter nào ta sẽ leak được file `flag.txt`
![image](https://hackmd.io/_uploads/B1uz1xBlkx.png)
* Vấn đề ở đây là không thể dùng các lệnh khác vì đã bị detect ở phía trên và ta cũng không thể nhập quá 1 lệnh.
* Đọc kỹ đoạn ```  except subprocess.TimeoutExpired:
                    return render_template('flag.html', txt=f'Timeout! Your key: {KEY}')
                except subprocess.CalledProcessError:
                    return render_template('flag.html', txt="Error!")``` Ta thấy đoạn này bắt ngoại lệ `TimeoutExpired` từ `subprocess`. Ngoại lệ này sẽ được ném ra khi một lệnh đang chạy quá thời gian tối đa mà bạn đã chỉ định cho nó. Khi xảy ra lỗi này, chương trình sẽ thực hiện trả key cho ta. tiến hành làm request time out bằng lệnh `sleep` 
![image](https://hackmd.io/_uploads/Hk6DleHxJg.png)
* Ta lấy được key và giờ có thể lấy flag bằng key này
![image](https://hackmd.io/_uploads/Sk_9gxBxJx.png)
> DH{9c73a5122e06f1f7a245dbe628ba96ba68a8317de36dba45f1373a3b9a631b92}
## 5. baby union
* Đọc tên bài cũng có thể biết được đây là dạng `SQLI` 
```python
import os
from flask import Flask, request, render_template
from flask_mysqldb import MySQL

app = Flask(__name__)
app.config['MYSQL_HOST'] = os.environ.get('MYSQL_HOST', 'localhost')
app.config['MYSQL_USER'] = os.environ.get('MYSQL_USER', 'user')
app.config['MYSQL_PASSWORD'] = os.environ.get('MYSQL_PASSWORD', 'pass')
app.config['MYSQL_DB'] = os.environ.get('MYSQL_DB', 'secret_db')
mysql = MySQL(app)

@app.route("/", methods = ["GET", "POST"])
def index():

    if request.method == "POST":
        uid = request.form.get('uid', '')
        upw = request.form.get('upw', '')
        if uid and upw:
            cur = mysql.connection.cursor()
            cur.execute(f"SELECT * FROM users WHERE uid='{uid}' and upw='{upw}';")
            data = cur.fetchall()
            if data:
                return render_template("user.html", data=data)

            else: return render_template("index.html", data="Wrong!")

        return render_template("index.html", data="Fill the input box", pre=1)
    return render_template("index.html")


if __name__ == '__main__':
    app.run(host='0.0.0.0')
```
* Code bị SQLI ở đoạn `cur.execute(f"SELECT * FROM users WHERE uid='{uid}' and upw='{upw}';")` thông qua `uid` không bị filter khi đưa vào câu lệnh SQL từ đó ta có thể thực hiện `SQLI` 
![image](https://hackmd.io/_uploads/HkdS67Hx1e.png)
* bước đầu tiên để crawl là kiểm tra số table trong database, ta biết được flag sẽ nằm trong bảng fake ở file `init.sql`
```sql
CREATE DATABASE secret_db;
GRANT ALL PRIVILEGES ON secret_db.* TO 'dbuser'@'localhost' IDENTIFIED BY 'dbpass';

USE `secret_db`;
CREATE TABLE users (
  idx int auto_increment primary key,
  uid varchar(128) not null,
  upw varchar(128) not null,
  descr varchar(128) not null
);

INSERT INTO users (uid, upw, descr) values ('admin', 'apple', 'For admin');
INSERT INTO users (uid, upw, descr) values ('guest', 'melon', 'For guest');
INSERT INTO users (uid, upw, descr) values ('banana', 'test', 'For banana');
FLUSH PRIVILEGES;

CREATE TABLE fake_table_name (
  idx int auto_increment primary key,
  fake_col1 varchar(128) not null,
  fake_col2 varchar(128) not null,
  fake_col3 varchar(128) not null,
  fake_col4 varchar(128) not null
);

INSERT INTO fake_table_name (fake_col1, fake_col2, fake_col3, fake_col4) values ('flag is ', 'DH{sam','ple','flag}');
```
![image](https://hackmd.io/_uploads/H1FeCXSgyl.png)
* Vậy database gồm 4 bảng. Dùng payload `Union Select table_name,null,null,null from information_schema.tables-- -` ta leak được bảng có tên là `onlyflag` và gồm các cột như sau
![image](https://hackmd.io/_uploads/ByCJ1NBe1x.png)
* tiến hành leak dữ liệu từ cột ra, do database chỉ có thể in ra 3 cột 1 lần nên ta có thể leak 2 lần để lấy được toàn bộ flag 
![image](https://hackmd.io/_uploads/SkZ2g4rgke.png)
> DH{57033624d7f142f57f139b4c9e84bd78da77b4406896c386672f0cbb016f5873

## 6. Type c-j
* Chương trình đơn giản vẫn là check input ta nhập vào 
```php
<html>
<head>
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
<title>Type c-j</title>
</head>
<body>
    <!-- Fixed navbar -->
    <nav class="navbar navbar-default navbar-fixed-top">
      <div class="container">
        <div class="navbar-header">
          <a class="navbar-brand" href="/">Type c-j</a>
        </div>
        <div id="navbar">
          <ul class="nav navbar-nav">
            <li><a href="/">Index page</a></li>
          </ul>
        </div><!--/.nav-collapse -->
      </div>
    </nav><br/><br/><br/>
    <div class="container">
    <?php
    function getRandStr($length = 10) {
        $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
        $charactersLength = strlen($characters);
        $randomString = '';
    
        for ($i = 0; $i < $length; $i++) {
            $randomString .= $characters[mt_rand(0, $charactersLength - 1)];
        
        }
        echo $randomString;
        return $randomString;
        

    }
    require_once('flag.php');
    error_reporting(0);
    $id = getRandStr();
    $pw = sha1("1");
    // POST request
    if ($_SERVER["REQUEST_METHOD"] == "POST") {
      $input_id = $_POST["input1"] ? $_POST["input1"] : "";
      $input_pw = $_POST["input2"] ? $_POST["input2"] : "";
      sleep(1);

      if((int)$input_id == $id && strlen($input_id) === 10){
        echo '<h4>ID pass.</h4><br>';
        if((int)$input_pw == $pw && strlen($input_pw) === 8){
            echo "<pre>FLAG\n";
            echo $flag;
            echo "</pre>";
          }
        } else{
          echo '<h4>Try again.</h4><br>';
        }
      }else {
      echo '<h3>Fail...</h3>';
     }
    ?> 
    </div> 
</body>
</html>
```
* Nhận vào hai đối số `input_1` và `input_2`.Nếu `input_id` là số nguyên, bằng `$id`, và dài 10 ký tự, cùng với `input_pw` là số nguyên, bằng `$pw`, và dài 8 ký tự, nó sẽ hiển thị thông báo `"ID pass."` và in ra giá trị của biến `$flag`.
* Vấn đề sẽ nằm ở phần ép kiểu thành `(int)` của `input_id`.
* Khi ta nhập id lấy từ random ví dụ là `gaaaaaaaaa`, PHP ép kiểu chuỗi này thành 0 vì nó không bắt đầu bằng số.
Ví dụ: (int)'gaaaaaaaa' sẽ trả về 0.
* Nếu `$id` cũng được tạo ra từ một chuỗi không chứa số hoặc bắt đầu bằng ký tự chữ, khi ép kiểu thành số nguyên, nó cũng trở thành 0.
Ví dụ: nếu id là abcdefg123, thì (int)$id cũng sẽ trả về 0.
Kiểm tra độ dài:
* Chuỗi gaaaaaaaaa có độ dài 10 ký tự, vì vậy điều kiện strlen($input_id) === 10 cũng đúng.
* Tiến hành enc `sha(1) = 356a192b7913b04c54574d18c28d46e6395428ab` nhưng do bài chỉ check 8 kí tự đầu nên ta chỉ cần lấy `356a192b`
> DH{33f2cac27f2cd193c31ac34b1b8bc99e2fbb731bd8a94c512c17d13dbf80794a}

## 7. random-test
* Bài này thực hiện việc brute id và mật khẩu khá thú vị 
```py
#!/usr/bin/python3
from flask import Flask, request, render_template
import string
import random

app = Flask(__name__)

try:
    FLAG = open("./flag.txt", "r").read()       # flag is here!
except:
    FLAG = "[**FLAG**]"


rand_str = ""
alphanumeric = string.ascii_lowercase + string.digits
for i in range(4):
    rand_str += str(random.choice(alphanumeric))

rand_num = random.randint(100, 200)


@app.route("/", methods = ["GET", "POST"])
def index():
    if request.method == "GET":
        return render_template("index.html")
    else:
        locker_num = request.form.get("locker_num", "")
        password = request.form.get("password", "")

        if locker_num != "" and rand_str[0:len(locker_num)] == locker_num:
            if locker_num == rand_str and password == str(rand_num):
                return render_template("index.html", result = "FLAG:" + FLAG)
            return render_template("index.html", result = "Good")
        else: 
            return render_template("index.html", result = "Wrong!")
            
            
app.run(host="0.0.0.0", port=8000)
```
* ta sẽ thực hiện nhập hai tham số `locker-num` và `password`. `  if locker_num != "" and rand_str[0:len(locker_num)] == locker_num:` đoạn đầu sẽ check `locker_num` không phải là chuỗi rỗng và chuỗi ta nhập vào có phải là chuỗi con của chuỗi `rand_str` không. Mấu chốt để giải quyết bài này là ở đây.
![image](https://hackmd.io/_uploads/BJ1HVm8lJx.png)
* Nếu ta cố đấm ăn xôi để brute 4 kí tự bao gồm chữ và số của `locker_num` và số từ 100-200 của `password` thì số payload ta phải gửi là gần 200 triệu (khá bất khả thi). Nhưng do việc chall check `locker_num` ta nhập vào xem có là chuỗi con của `rand_str` không chứ k phải là toàn bộ chuỗi và vòng if check `locker_num` trước so với `password` nên ta có thể thực hiện brute từng kí tự của `locker_num` trước với payload 
![image](https://hackmd.io/_uploads/S1uJDQIxkg.png)
![image](https://hackmd.io/_uploads/HJhYIXUgyl.png)
* Ta đã check được kí tự đầu của `locker_num[0] = 'a'`. cứ lần lượt đổi payload từ 0-3 ta sẽ lấy được `locker_num = '05x0'` để xác nhận lại ta sẽ thử nhập vào server 
![image](https://hackmd.io/_uploads/HkxFvmIe1e.png)
* Giờ ta chỉ cần brute `password` với cận là từ 100-200
![image](https://hackmd.io/_uploads/H1FpP7Igkg.png)
![image](https://hackmd.io/_uploads/rJjxOXIxyx.png)
> DH{2e583205a2555b8890d141b51cee41379cead9a65a957e72d4a99568c0a2f955}

## 8. Simple-SSTI
![image](https://hackmd.io/_uploads/r1Y0W0wxkx.png)
* Server có tác dụng check path nếu sai sẽ trả về `404 not found` 
```python
#!/usr/bin/python3
from flask import Flask, request, render_template, render_template_string, make_response, redirect, url_for
import socket

app = Flask(__name__)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

app.secret_key = FLAG


@app.route('/')
def index():
    return render_template('index.html')

@app.errorhandler(404)
def Error404(e):
    template = '''
    <div class="center">
        <h1>Page Not Found.</h1>
        <h3>%s</h3>
    </div>
''' % (request.path)
    return render_template_string(template), 404

app.run(host='0.0.0.0', port=8000)
```
* Xem qua source ta thấy được chall bị dính `SSTI` ở đoạn ` return render_template_string(template)` khi return luôn template của user nhập vào mà không filter. Ta tiến hành fuzzing thử, nhập vào payload {{7*7}}
![image](https://hackmd.io/_uploads/HyfuMCvxke.png)
* Server đã thực thi input ta như một `template_string` và trả về 49, lúc này mình sẽ nhập {{7*'7'}} để xem thử loại template server đang dùng là gì 
![image](https://hackmd.io/_uploads/SydnzAweke.png)
* Kết quả trả về `7777777` do đó ta biết được template đang dùng là `jinja2`. Tiến hành đọc file `flag.txt` bằng `jinja2` 
`{{ url_for.__globals__.current_app.config }}`
* Lệnh này sẽ hiển thị các cấu hình hiện tại của `Flask`, bao gồm các biến quan trọng như `SECRET_KEY`, cấu hình `DEBUG, ENV,` hoặc các thiết lập liên quan đến cơ sở dữ liệu.
![image](https://hackmd.io/_uploads/HJjEQRvlJl.png)
* Ta lấy được flag
>  DH{6c74aac721d128c637eab3f11906a44b}
