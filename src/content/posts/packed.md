---
title: Learning about Packed File (1)
published: 2024-05-05
description: Learning about Packed File part 1
tags: [Portswigger, Pentest]
category: 'Learning'
draft: false 
image: "/assets/post_image/pack.png"
---


# Learning: Tổng Quan về File Packed


## I. File Packed là gì
* Định nghĩa đơn giản về tập tin packed đó là một tập tin được ẩn mã thực thi gốc của chương trình và lưu lại bằng cách áp dụng các kỹ thuật nén hoặc mã hóa để tránh không bị reverse một cách dễ dàng. Bên cạnh đó nó cũng được chèn thêm một đoạn Stub hay một section được gọi là **`Packed Data`**, để khi thực thi chương trình, đoạn stub này sẽ nhận code đã được mã hóa, giải mã nó trong bộ nhớ, lưu code giải mã vào trong bất kỳ section nào hoặc là chính nó và cuối cùng nhảy tới vùng code này để thực thi (đó chính là code gốc ban đầu của file).

    ![packing (1)](https://hackmd.io/_uploads/ByNYIi1ZA.png)

* Ta có thể thấy ảnh trên có sự khác biệt lớn giữa 1 file thực thi original và 1 packed file. Packed file đã loại bỏ các Text, Data và RSRC section thay vào đó chứng sẽ được pack lại thành **`Packed Original Sections`** và đồng thời packed file cũng push thêm 1 vùng là **OEP(Original Entry Point)**, **Entry Point(EP)** của chương trình sẽ được chuyển hướng vào vùng code này. `OEP` là mấu chốt để chúng ta unpack và lấy ra được packed data từ file PE đã bị packed.


## II. Mục đích của việc Pack File thực thi là gì?

* Hầu hết các Packed file được dùng để che giấu source code gốc nhằm để cách Cracker/Reverser khó khăn trong việc dịch ngược mã nguồn và lấy thông tin từ file thực thi của họ. Và một trong những cách hay dùng đối với việc pack file đó chính là dùng để tăng khả năng chống trọi lại **`AV`** của các **`Malware`**. Chúng sẽ gây khó khăn hơn cho các White hat Hacker để có thể bẻ khóa và hiểu được cách hoạt động của **`Malware`** 

* Thông thường khi một file bị packed khi bị **`AV`** scan thì đa phần các **`AV`** sẽ quét theo tuyến tính, từ **`DOS MZ Header`**, **`PE Header`** rồi các section trong section table. Mà section table của một file PE bị packed như đã nói ở trên sẽ được chèn thêm rất nhiều trash sections, các sections này với các phần mềm **`AV`** thì nó vô hại. Trong khi section thực sự có hại nằm ở Packed Data, dữ liệu được nén trong Packed Data chỉ được giải nén khi chạy chương trình.

## III. Cách Pack 1 file như thế nào?

* Có rất nhiều biến thể của các trình packers, và đa phần trong số chúng là các trình protectors, sử dụng các kĩ thuật như hủy bảng IAT hay import table, hủy thông tin về Header của file. Bổ sung thêm các cơ chế anti-debugger để tránh việc unpack và khôi phục lại tập tin ban đầu.

* Ví dụ đơn giản nhất mà các chuyên gia hay sử dụng để minh họa về packer đó chính là UPX. Packer này không áp dụng các Anti_debugger, hay các tricks nào khác, tuy nhiên nó lại giúp ta khởi đầu với những kĩ năng đơn giản nhất trong quá trình thực hiện unpacking. 

> **Link để tải UPX mình sẽ [để ở đây](https://github.com/upx/upx/releases/tag/v3.96)**


![Screenshot 2024-04-19 154911](https://hackmd.io/_uploads/SJrdFWl-0.png)


* Ta sẽ được một terminal thế này khi load vào cmd, để ví dụ cho một file packed thì mình sẽ dùng chương trình quản lý sinh viên **`Student_Management.cpp`**. Link source code của chương trình mình sẽ [để ở đây](https://github.com/AhmadFaraz-crypto/Student-Management-System/blob/master/SourceCode.cpp)

    ![image](https://hackmd.io/_uploads/Hkdy0MBzR.png)

* Chương trình đơn giản sẽ check số sinh viên và điểm của sinh viên như các chương trình quản lý sinh viên khác nên ta sẽ không đề cấp đến phần hoạt động chính của chương trình.

* Mình sẽ compile chương trình thành file exe và load IDA để check xem main và các sections có hoạt động ok hay không

    ![Screenshot 2024-04-19 211154](https://hackmd.io/_uploads/H1fFSbxWR.png)

* Ta thấy hàm main đã đầy đủ thông tin, Hàm cout trông có vẻ kinh dị vì mình nghĩ do source code của ta được code bằng cpp nhưng khi compile thành .exe và load vào IDA thì lại được IDA compile từ binary lên ASM và compile thành C, và do C dùng printf có sự khác biệt về cấu trúc nên mới kinh dị như vậy :v, nhưng dù sau cũng chỉ là chuyện vặt không ảnh hưởng đến quá trình phân tích của ta

    ![Screenshot 2024-04-19 214525](https://hackmd.io/_uploads/HJ9VU-lbA.png)

* Đây là phần **`Subview Segmentation`** của file .exe. Ta đều thấy có đủ 4 sections đặc trưng của một file PE như ta đã nói ở trên đó là **`Text Section (.text)`**, **`Data Sections`** gồm các sections là `r, p, x.data,....` và **`RSRC Section (.rsrc)**. 


* Vậy đã check xong file Student_Management.exe tiếp đến ta sẽ tiến hành pack file bằng upx vầ command như sau `upx -o File_Packed.exe File.exe`

    ![Screenshot 2024-04-19 212527](https://hackmd.io/_uploads/S1zv1bebR.png)

* Thông báo hiển thị ta đã packed xong file mình sẽ load luôn file này vào IDA 

    ![image](https://hackmd.io/_uploads/SJzSuZxZR.png)

* Ta thấy sơ bộ thì các hàm call trong main cũng như các class đã được set trong source đã mất tiu, và chúng ta đang dừng ở `EP`

    ![Screenshot 2024-04-19 214519](https://hackmd.io/_uploads/HyCO_WxZ0.png)

* Phần **`Segmentation`** cũng đã biến mất các sections quan trọng của file như .text, đồng thời ta có thêm 2 sections là `UPX0` và `UPX1`. Hãy check phần EP của cả hai file ở từng sections ta sedx thấy như sau 

    * Đôí với file .exe: 

        ![image](https://hackmd.io/_uploads/r1IG3bM-0.png)
        * Ta thấy rằng **`Virtual size`** = 0x3108 byte còn **`Section size in file (raw size)`** = 0x3200 byte, ta thấy độ chênh lệch kích thước giữa hai size này là không lớn tầm khoảng 100 byte 

    * Đối với file packed:

        ![image](https://hackmd.io/_uploads/ByGp3bf-0.png)
        * Còn đối với file packed thì phần **`Virtual size`** và phần **`Raw size`** chênh lệch kích thước rất lớn


* Có sự chênh lệch lớn như vậy là do **`UPX`** đã cấu trúc file thực thi sao cho không cần phải lưu trữ `Packed Data` thực sự trong file. Thay vào đó, nó được giải nén và thực thi trực tiếp từ **`section UPX0`** trong bộ nhớ khi file thực thi được chạy. Do đó, khi phân tích file thực thi trên ổ đĩa, **`Raw size`** của **`section UPX0`** có thể là 0 bytes, nhưng khi file thực thi được chạy, dữ liệu nén sẽ được giải nén và sử dụng trong bộ nhớ, dẫn đến **`Virtual`** size lớn.

*  Còn trong trường hợp một file .exe không bị nén, kích thước của các section trong file ở **`Raw size`** với **`Virtual size`** sẽ không chênh lệch nhiều. Điều này có nghĩa là dữ liệu trong các section sẽ được lưu trữ trực tiếp trong file thực thi và được load vào bộ nhớ một cách gần như không đổi khi file thực thi được chạy.

* Khi ta quăng cả hai file vào PeStudio để chạy thử ta sẽ check được sâu hơn các artifact như sau. Link tải PeStudio mình sẽ [để ở đây](https://www.winitor.com/download2)

    * File exe:

    ![Screenshot 2024-04-21 103449](https://hackmd.io/_uploads/rJAu1fzW0.png)

    * File packed
    ![image](https://hackmd.io/_uploads/ByN_JfGZC.png)

* Ta nhìn ở phần `Entropy` thì thấy file packed có độ `Entropy` cao hơn nhiều so với file .exe. Nói về **`Entropy`** ta có thể hiêu đơn giản như sau, thì đó chính là độ rối của file binary, nếu các opcode sắp xếp không giống chương trình bình thường mà được sắp xệp một cách hỗn loạn thì **`Entropy`** sẽ càng cao.

* Trong phân tích Malware nếu một file có mức độ entropy cao trong các section thì có thể giúp phát hiện ra các phần đã packed hoặc mã hóa. Tuy nhiên, việc xác định xem một section chứa **`Packed Data`** hay không dựa trên giá trị **`Entropy`** một cách đơn thuần có thể không chính xác 100%, vì có thể có các kỹ thuật làm giảm **`Entropy`** mà không cần phải sử dụng kỹ thuật nén hoặc mã hóa. 

## IV. Tìm hiểu file packed và cách tìm Original Entry Point

![Screenshot 2024-04-26 195841](https://hackmd.io/_uploads/r109O7KbR.png)

* Nhìn vào tổng quan hai section là **`UPX0`** và **`UPX1`**, ta có thể thấy **`UPX0`** start ở địa chỉ `0x14001000` trùng với .text bên file gốc. Khi chúng ta làm việc với một chương trình bị packed, ta sẽ không biết OEP của nó ở đâu, do đó chúng ta sẽ phải áp dụng các kĩ thuật để tìm ra OEP.

    ![image](https://hackmd.io/_uploads/r1vqAj9-R.png)

* Nhìn vào đoạn start của chương trình ta có thể thấy lệnh ` lea     rsi, byte_140023025` `lea     rdi, [rsi-22025h]` thực hiện việc lấy địa chỉ từ `0x140023025` sau đó lưu lại kết quả vào `0x140001000` (do 0x140023025 - 0x22025 = 140001000) địa chỉ này trùng với UPX0 ta có từ đó ta có thể đoán được rằng UPX0 sẽ là nơi chưa Packed Data hay OEP ta cần tìm, nhưng khi jump đến địa chỉ start của UPX0 mình lại không nhận được tham chiếu nào khác ngoài Header, nên mình quyết định rà soát start một lần nữa cho đến khi thấy đoạn stub này
    ![image](https://hackmd.io/_uploads/rykBvtMM0.png)
    
     
* Đây là lệnh `jump near ptr qword_1400013F0`,`Jmp near`là một lệnh nhảy trực tiếp đến địa chỉ, do vậy nó sẽ nhảy thẳng đến 0x1400013F0. Mình sẽ đặt bp ở đọan jump này


    ![image](https://hackmd.io/_uploads/S1MXutfG0.png)

* Địa chỉ trên chính là phần code OEP gốc đã encrypted bởi UPX vì sao mình biết như vậy thì khi debug cùng lúc lệnh call ở `RIP` được gọi thì ta cũng được nhảy tới flow chính của chương trình cũng như main của .exe

* Ta đã đến được bước thứ nhất và cũng như là bước quan trọng nhất của việc Unpack 1 file đó chính là tìm OEP, ở phần sau ta sẽ tìm hiểu thêm về 2 bước còn là dump file và fix IAT để unpack file:v
