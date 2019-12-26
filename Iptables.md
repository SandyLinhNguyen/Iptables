# Iptables
## Tổng quan
Iptables là một hệ thống tường lửa (Firewall) được cấu hình, tích hợp mặc định trong hầu hết các phiên bản của hệ điều hành linux (CentOS, Ubuntu,...). Iptables do Netfilter Organization viết ra để tăng tính năng bảo mật trên hệ thống Linux. Iptables cung cấp các tính năng sau:
```txt
Tích hợp tốt với kernel của Linux
Có khả năng phân tích package hiệu quả
Lọc package dựa vào MAC và một số cờ hiệu trong TCP Header
Cung cáp chi tiết các tuỳ chọn để ghi nhận sự kiện hệ thống
Cung cấp kỹ thuật NAT
Có khả năng ngăn chặn một số cơ chế tấn công theo kiểu DoS
```

## Cách sử dụng
### Cài đặt và khởi động iptables(Centos 7)
Sử dụng các tập lệnh sau để cài đặt:
```bash
# install iptables
sudo yum install -y iptables-services
# start iptables service
sudo systemctl start iptables
# auto start iptables service
systemctl enable iptables
```
Lưu ý: Nếu cài trên Ubuntu, cần vô hiệu hoá ufw để tránh xung đột ufw và iptables vì cả 2 đều là tường lửa mặc định: ```ufw disable```

## Các khái niệm cơ bản

### Các bảng trong Iptables
Iptables có 5 bảng với mục đích và thứ tự xử lý khác nhau. Thứ tự xử lý các gói tin được mô tả cơ bản như sau:
- Filter Table: Bảng này được sử dụng nhiều nhất trong Iptables, bảng này quyết định xem gói tin được đi tiếp hay bị chặn lại, và đây cũng là chức năng quan trọng nhất trong Iptables
- Nat table: được dùng để NAT(Network Address Tránlation), khi các gói tin đi vào bảng này, gói tin sẽ được kiểm tra xem có cần thay đổi và sẽ thay đổi địa chỉ nguồn, đích của gói tin. Bảng này được sử dụng khi có một gói tin từ một connection mới gửi tới hệ thống, các gói tin tiếp theo của connection này sẽ áp dụng rule và xử lý tương tự như gói tin đầu tiên mà không cần phải đi qua bảng NAT nữa.
- Mangle Table: Dùng để điều chỉnh một số trường trong IP Header như TTL (Time to live), ToS (type of service) dùng để quản lý chất lượng dịch vụ (Quality of Service)... hoặc dùng để đánh dấu các gói tin để xử lý thêm trong các bảng khác
- Raw Table: Theo default, Iptables sẽ lưu trạng thái kết nối của các gói tin, tính năng này cho phép iptables xem các gói tin rời rạc là một kết nối, một session chung để dễ quản lý. Tính năng theo dõi này được sử dụng ngay từ khi gói tin được gởi tới hệ thống trong bảng raw
- Security Table: Bảng security dùng để đánh dấu policy của SELinux lên các gói tin, các dấu náy ẽ ảnh hưởng đến cách thức xử lý của SElinux hoặc của các máy khác trong hệ thống có áp dụng SELinux. Bảng này có thể đánh dấu theo từng gói tin hoặc theo từng kết nối

### Các chain trong table
Mỗi table đều có một số chain của riêng mình, sau đây là bảng cho biết các chain thuộc mỗi table

| TABLES/CHAIN | PREROUTING | INPUT | FORWARD | OUTPUT | POSTROUTING |
|:------------:|:----------:|:-----:|:-------:|:------:|:-----------:|
|     RAW      |     X      |       |         |   X    |             |
|    MANGLE    |     X      |   X   |    X    |   X    |      X      |
|    FILTER    |            |   X   |    X    |   X    |             |
|   SECURITY   |            |   X   |    X    |   X    |             |
|  NAT(DNAT)   |     X      |       |         |   X    |             |
|  NAT(SNAT)   |            |   X   |         |        |      X      |

Giới thiệu cơ bản qua các chain như sau:
- INPUT: Chain này dùng để kiểm soát hành vi của các kết nối đến máy chủ
- FORWARD: Chain này dùng cho các kết nối chuyển tiếp sang một máy chủ khác
- OUTPUT: Chain này sẽ xử lý các kết nối ra ngoài.
- PREROUTING: Header của gói tin sẽ được điều chỉnh tại đây trước khi việc routing được diễn ra
- POSTROUTING: Header của gói tin sẽ được điều chỉnh tại đây sau khi việc routing được diễn ra

Mặc định thì các chain này sẽ không chứa bất kỳ một rule nào, tuy nhiên mỗi chain đều có một policy mặc định nằm ở cuối chain, policy này có thể là ACCEPT hoặc DROP, chỉ khi gói tin đã đi qua hết tất cả các rule ở trên thì gói tin mới gặp policy này

### Dùng lệnh iptables -L để liệt kê các rules
![iptables -L](../Iptables/images/iptablesL.png)

Các rule là tập điều kiện và hành động tương ứng để xử lý gói tin. Mỗi chain sẽ chứa rất nhiều rule, gói tin được xử lý trong một chain sẽ đowjc so với lần lượt từng rule trong chain này.
Rule có thể dựa trên protocol, địa chỉ nguồn/đích, port nguồn/đích, card mạng, header gói tin, trạng thái kết nối... Dựa trên những nhiều kiện này, ta có thể tạo ra một tập rule phức tạp để kiểm soát luồng dữ liệu vào ra hệ thống
Mỗi rule sẽ được gắn một hành động để xử lý gói tin, hành động này có thể là một trong những hành động sau:
- ACCEPT: Gói tin sẽ được chuyển tiếp sang bảng kế tiếp
- DROP: Gói tin/kết nối sẽ bị huỷ, hệ thống không thực thi bất kỳ lệnh nào khác
- REJECT: Gói tin sẽ bị huỷ, hệ thống sẻ gởi ại 1 gói tin báo lỗi ICMP - Destination port unreachable
- LOG: Gói tin khớp với rule sẽ được ghi log lại
- REDIRECT: Chuyển hướng gói tin sang một proxy khác
- MIRROR: Hoán đổi địa chỉ nguồn, đích của gói tin trước khi gởi gói tin này đi
- QUEUE: Chuyển gói tin tới chương trình của người dùng qua một module của kernel

### Các trạng thái kết nối
Dưới đây là những trạng thái của các kết nối:
- NEW: Khi có một gói tin mới được gởi tới và không nằm trong bất kỳ connection nào hiện có, hệ thống sẽ khởi tạo một kết nối mới và gắn nhãn NEW cho kết nối này. Nhãn này dùng cho cả TCP và UDP
- ESTABLISHED: Kết nối được chuyển từ trạgn thái NEW sang ESTABLISHED khi máy chủ nhận được phản hồi
- RELATED: Gói tin được gởi tới không thuộc về một kết nối hiện có nhưng có liên quan đến một kết nối đang có trên hệ thống. Đây chỉ là một kết nối phụ hỗ trợ cho kết nối chính
- INVALID: Gói tin được đánh dấu INVALID khi gói tin này không có bất ký quan hệ gì với các kết nối đang có sẵn, không thích hợp để khởi tạo một kết nối mới hoặc đơn giản là không thể xác định được gói tin này, không tìm được kết quả trong bảng định tuyến
- UNTRACKED: Gói tin có thể được gắn nhãn UNTRACKED nếu gói tin này đi qua bảng raw và được xác định là không cần theo dõi gói này trong bảng connection tracking
- SNAT: Trạng thái này được gán cho các gói tin mà địa chỉ nguồn đã bị NAT, được dùng bởi hệ thống connection tracking để biết khi nào cần thay đổi lại địa chỉ các gói tin khi trả về
- DNAT: Trạng thái này được gán cho các gói tin mà địa chỉ đích đã bị NAT, được dùng bởi hệ thống connection tracking để biết khi nào cần thay đổi địa chỉ cho các gói tin gởi đi

Các trạng thái này giúp người quản trị tạo ra các rule cụ thể và an toàn hơn cho hệ thống

### Cách sử dụng iptables để mở port
Để mở port trong Iptables, chúng ta cần chèn chuỗi ACCEPT PORT. Cấu trúc lệnh để mở port:
```bash
iptables -A INPUT -p tcp -m tcp --dport xxx -J ACCEPT
hoặc
iptables -I INPUT -p tcp -m tcp --dport xxx -j ACCEPT
# Trong đó, A tức là Append, chèn vào cuối của chuỗi INPUT, còn I là để chỉ định số dòng rule muốn chèn. Để tránh xung đột với rule gốc, chúng ta nên chèn rule vào đầu bằng option -I
```
Ví dụ: 
```bash
# Để truy cập VPS qua SSH, chúng ta cần mở port SSH 22, ở đây chúng ta thiết lập cho phép kết nối SSH ở bất cứ thiết bị nào và ở bất cứ đâu:
iptables -I INPUT -p tcp -m tcp --dport 22 -j ACCEPT
# Cho phép kết nối VPS qua SSH duy nhất từ một địa chỉ IP xác định
iptables -I INPUT -p tcp -s xxx.xxx.xxx.xxx -m tcp --dport 22 -j ACCEPT
```
Hoặc:
```bash
# Cho phép user sử dụng SMTP servers qua port mặc định 25 và 465:
iptables -I INPUT -p tcp -m tcp --dport 25 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 465 -j ACCEPT
# Để user đọc email trên server, cần mở port POP3(mặc định là 110 và 995)
iptables -A INPUT -p tcp -m tcp --dport 110 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 995 -j ACCEPT
# Cho phép giao thức IMAP mail protocol(mặc định là 143 và 993)
iptables -I INPUT -p tcp -m tcp --dport 143 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 993 -j ACCEPT
```
Sau khi thiết lập xong, chúng ta cần lưu thiết lập Iptables nếu không các thiết lập sẽ mất khi chúng ta reboot hệ thống:
```bash
service iptables save
```
