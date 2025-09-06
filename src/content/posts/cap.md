---
title: HTB CAP
published: 2025-08-26
tags: [HTB, Linux, Easy]
category: Writeups
author: Trohan0x00
description: Writeup about Cap Machine
image: '/assets/post_image/cap.png'
draft: true
---

# HTB Labs: Cap
## I. User Flag

Đây sẽ làm một writeup nhanh với machine này, vì nhìn chung đây là một machine tương đối đơn giản. Scan bằng nmap ta thấy có 3 services
![image](https://hackmd.io/_uploads/BkA07giFxl.png)
Mình sẽ tiến hành check ở phía web server trước
![image](https://hackmd.io/_uploads/HJe4EloKxe.png)
đây có lẽ là một web về giải pháp hoặc cung cấp các dịch vụ quan sát mạng hay đại loại thế. Vì check ở phần `Security Snapshot` ta có thể tải về bản file `pcap` của user `Nathan` mà ta đang login
Nếu ta để ý kỹ ở phía `URL`, khả năng sẽ là một lỗ hỏng `idoor`, vì khi ta thực hiện đổi số ở phần data thì thấy được số packet cũng thay đổi theo
![image](https://hackmd.io/_uploads/SyIUBeiKle.png)
Sau một lúc check thì mình thấy ở phái `/data/0` có số lượng packet lớn nhất. Mình sẽ follow và download bản `pcap` này
Sau khi filter với `ftp` thì mình lấy đuợc credential của `Nathan`
![image](https://hackmd.io/_uploads/SybFdxoYxl.png)
login fpt với credential `nathan:Buck3tH4TF0RM3!` thì ta lấy được `user_flag`
![image](https://hackmd.io/_uploads/H1-GFliFgl.png)
> cbd2a96465af9f33d828a4f84e67a064


## II. Root flag
Vẫn với credential cũ, ta có thể login vào `ssh` server để thực hiện việc leo quyền. Ta dùng `getcap -r / 2>/dev/null` để check tất cả các file có `capabilities` 
![image](https://hackmd.io/_uploads/rJ63CejYge.png)
Ta thấy được ngay file `python3.8` có thể lợi dụng để leo quyền, cụ thể ta sẽ import os vào với payload 
```python 
import os
os.setuid(0)
os.system('/bin/bash')
```
payload sẽ mở một shell bash lên với uid là 0 `root`. 
![image](https://hackmd.io/_uploads/SkIXJWitel.png)
> b96acca6f4fcffcf28b15eba9f1b82a6