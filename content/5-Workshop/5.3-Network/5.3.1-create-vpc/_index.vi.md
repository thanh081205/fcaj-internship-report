---
title : "Tạo VPC, subnet và định tuyến"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.3.1 </b> "
---

Ở bước này chúng ta tạo VPC, ba subnet, một Internet Gateway và hai route table quyết định subnet nào được ra Internet.

#### Tạo VPC và subnet

Mở console **VPC**, chọn **Create VPC**.

![tạo vpc](/images/5-Workshop/5.3-Network/create-vpc.png)

Thiết lập như sau:

| Mục | Giá trị |
|---|---|
| IPv4 CIDR block | `10.0.0.0/16` |
| IPv6 CIDR block | No IPv6 CIDR block |
| Tenancy | Default |
| DNS hostnames | Enabled |
| DNS resolution | Enabled |

**DNS hostnames** và **DNS resolution** đều phải được bật. VPC Endpoint dựa vào Private DNS để ghi đè tên miền công khai của dịch vụ AWS bằng địa chỉ IP riêng, và Private DNS sẽ lặng lẽ không hoạt động nếu một trong hai tùy chọn này bị tắt.

Chọn **Create VPC** và chờ trình hướng dẫn chạy xong.

Tạo subnet: hai public subnet là `10.0.1.0/24` và `10.0.2.0/24` và một private subnet là `10.0.10.0/24`.

![subnet 1](/images/5-Workshop/5.3-Network/subnet1.png)

![subnet 2](/images/5-Workshop/5.3-Network/subnet2.png)

![subnet 3](/images/5-Workshop/5.3-Network/subnet3.png)

#### Tạo internet gateway cho vpc vừa tạo

![igw](/images/5-Workshop/5.3-Network/igw.png)

#### Tạo route table

Tạo một route table cho pulbic subnet, có route trỏ đến internet gateway vừa tạo. 

![public route table](/images/5-Workshop/5.3-Network/public-rt.png)

Tạo một route table cho private subnet, có route trỏ đến endpoint dùng để truy cập **S3**.

![private route table](/images/5-Workshop/5.3-Network/private-rt.png)



