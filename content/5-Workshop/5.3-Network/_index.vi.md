---
title : "Xây dựng nền tảng mạng"
date : 2024-01-01
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

Mọi tài nguyên trong các phần còn lại của workshop đều nằm bên trong mạng mà chúng ta xây dựng ở đây. Database, cache, message broker và cả ba ECS task đều đặt trong private subnet không có đường ra Internet; thứ public là application load balancer.

#### Nguyên tắc thiết kế

**Mặc định là private.** Compute và dữ liệu nằm trên subnet không có route tới Internet Gateway. Không thành phần nào của hệ thống cần được truy cập từ bên ngoài ngoại trừ load balancer, nên không thành phần nào khác được cấp đường đi công khai.

**Không dùng NAT Gateway.** ECS task vẫn cần kéo image từ ECR, đẩy log lên CloudWatch và đọc tham số từ Systems Manager. Thay vì đẩy lưu lượng đó ra ngoài qua NAT Gateway rồi vòng ngược lại vào AWS, chúng ta đặt **VPC Endpoint** ngay bên trong VPC để lưu lượng không bao giờ rời khỏi mạng AWS.

**Security group tham chiếu security group.** Rule được viết dưới dạng "cho phép port 5432 từ security group của ECS task", không bao giờ viết "cho phép port 5432 từ 10.0.10.0/24". Khi IP của task thay đổi — điều xảy ra sau mỗi lần deploy trên Fargate — rule vẫn hoạt động bình thường.

#### Quy hoạch dải địa chỉ

| Subnet | CIDR | Availability Zone | Loại | Chứa gì |
|---|---|---|---|---|
| Public A | `10.0.1.0/24` | `ap-southeast-1a` | Public | Application Load Balancer |
| Public B | `10.0.2.0/24` | `ap-southeast-1b` | Public | Application Load Balancer |
| Private A | `10.0.10.0/24` | `ap-southeast-1a` | Private | ECS task, RDS, ElastiCache, Amazon MQ, VPC Endpoint |

VPC dùng dải `10.0.0.0/16`, còn dư rất nhiều chỗ để bổ sung subnet về sau.

Sở dĩ có hai public subnet là vì Application Load Balancer bắt buộc phải có subnet ở ít nhất hai Availability Zone — nếu chỉ chọn một AZ thì AWS sẽ từ chối tạo. Bản thân ALB là tài nguyên duy nhất nằm ở đó.

{{% notice note %}}
Toàn bộ phần còn lại được đặt trong một private subnet duy nhất ở `ap-southeast-1a`. Đây là lựa chọn đơn giản hóa có chủ đích nhằm giữ chi phí workshop ở mức chấp nhận được: thêm một AZ nữa đồng nghĩa với một bộ VPC Endpoint thứ hai và database Multi-AZ, làm chi phí theo giờ tăng gần gấp đôi. Đổi lại, nếu AZ đó gặp sự cố thì toàn hệ thống ngừng hoạt động. Một hệ thống production thật sự nên trải trên cả hai AZ.
{{% /notice %}}

![sơ đồ mạng](/images/5-Workshop/5.3-Network/network-diagram.png)

#### Các luồng lưu lượng

Sau khi hoàn thành phần này, lưu lượng chỉ di chuyển theo đúng ba cách:

+ **Vào từ bên ngoài** — Internet đi tới ALB trong public subnet, ALB chuyển tiếp tới ECS task trong private subnet. Không thành phần nào khác nhận kết nối từ bên ngoài.
+ **Nội bộ** — các ECS task nói chuyện với nhau và với tầng dữ liệu hoàn toàn bên trong private subnet, qua địa chỉ IP riêng.
+ **Đi ra tới dịch vụ AWS** — ECS task truy cập ECR, CloudWatch Logs, Systems Manager, KMS, Amazon MQ và S3 thông qua VPC Endpoint, vốn phân giải thành địa chỉ IP riêng bên trong VPC.

Không có luồng thứ tư. Một task không thể truy cập tới địa chỉ tùy ý trên Internet — đây là một phần đáng kể trong thế trận bảo mật, nhưng cũng là điều cần nhớ nếu sau này bạn thêm một phụ thuộc vào API bên ngoài.

#### Nội dung

1. [Tạo VPC, subnet và định tuyến](5.3.1-create-vpc/)
2. [Tạo các security group](5.3.2-security-groups/)
3. [Tạo VPC Endpoint](5.3.3-vpc-endpoints/)
4. [DNS và tầng edge CloudFront](5.3.4-dns-cdn/)
