---
title : "Tạo VPC Endpoint"
date : 2024-01-01
weight : 3
chapter : false
pre : " <b> 5.3.3 </b> "
---

Private subnet không có đường ra Internet, nhưng các ECS task đặt trong đó vẫn cần truy cập một số dịch vụ AWS. VPC Endpoint giải quyết chuyện này bằng cách đặt một điểm vào cho từng dịch vụ ngay bên trong VPC.

#### Task thực sự cần những gì

Trước cả khi container khởi động, Fargate phải xác thực với ECR, tải các layer của image về, và tạo log stream. Nếu bất kỳ lời gọi nào trong số đó không hoàn tất, task sẽ thất bại ngay ở giai đoạn provisioning, với thông báo lỗi chỉ về phía mạng chứ không phải về ứng dụng của bạn. Khi đã chạy, container còn đọc cấu hình từ Systems Manager và kết nối tới message broker.

Từ đó ta có danh sách cụ thể sau:

| Endpoint | Loại | Dùng để |
|---|---|---|
| `com.amazonaws.ap-southeast-1.ecr.api` | Interface | Xác thực với ECR và phân giải image manifest |
| `com.amazonaws.ap-southeast-1.ecr.dkr` | Interface | Docker Registry API dùng để kéo image |
| `com.amazonaws.ap-southeast-1.s3` | **Gateway** | Tải chính các layer của image |
| `com.amazonaws.ap-southeast-1.logs` | Interface | Đẩy log container lên CloudWatch |
| `com.amazonaws.ap-southeast-1.ssm` | Interface | Đọc cấu hình từ Parameter Store |
| `com.amazonaws.ap-southeast-1.kms` | Interface | Giải mã các tham số `SecureString` |
| `com.amazonaws.ap-southeast-1.ssmmessages` | Interface | ECS Exec — mở shell vào task đang chạy |
| `com.amazonaws.ap-southeast-1.ec2messages` | Interface | Kênh điều khiển của ECS Exec |
| `com.amazonaws.ap-southeast-1.mq` | Interface | API quản lý của Amazon MQ |

{{% notice note %}}
Việc kéo một image cần tới **ba** endpoint, không phải một. `ecr.api` lo phần xác thực và metadata, `ecr.dkr` phục vụ giao thức registry, còn bản thân các layer thì được lưu trên S3 — nên endpoint S3 là bắt buộc để kéo image, kể cả khi ứng dụng của bạn không hề đụng tới bucket nào. Thiếu endpoint S3 chính là nguyên nhân kinh điển khiến task xác thực ECR thành công rồi đứng im ở bước tải image.
{{% /notice %}}

Endpoint cho S3 thuộc loại **Gateway** chứ không phải Interface. Gateway endpoint hoạt động bằng cách thêm một route vào route table thay vì tạo network interface, và nó **miễn phí** — điều đáng lưu ý khi đọc phần chi phí bên dưới.

#### Tạo security group cho endpoint

Interface endpoint thực chất là các network interface nằm trong subnet của bạn, nên chúng cần một security group. Tạo thêm một group nữa:

| Tên | Port | Source |
|---|---|---|
| `vsp-vpce-sg` | 443 | `vsp-ecs-tasks-sg` |
| `vsp-vpce-sg` | 443 | `vsp-qdrant-sg` |

Mọi API dịch vụ của AWS đều chạy trên HTTPS, nên chỉ cần mở port 443. Cả hai security group của task đều được liệt kê làm source vì task Qdrant cũng phải kéo image từ ECR và ghi log.

#### Tạo các interface endpoint

Với từng endpoint trong tám interface endpoint: console **VPC** → **Endpoints** → **Create endpoint**.

![tạo interface endpoint](/images/5-Workshop/5.3-Network/interface-endpoint.png)

| Mục | Giá trị |
|---|---|
| Type | AWS services |
| VPC | VPC đã tạo ở 5.3.1 |
| Subnets | chỉ chọn Private subnet A (`10.0.10.0/24`) |
| Enable DNS name | **Tích chọn** |
| Security group | `vsp-vpce-sg` |
| Policy | Full access |

**Enable DNS name** chính là tùy chọn khiến toàn bộ cơ chế này trở nên trong suốt với ứng dụng. Khi Private DNS được bật, một request tới `ecr.ap-southeast-1.amazonaws.com` từ bên trong VPC sẽ phân giải thành IP riêng của endpoint thay vì IP công khai. Container và AWS CLI của bạn không cần sửa cấu hình gì cả — chúng vẫn gọi đúng tên dịch vụ như bình thường, còn lưu lượng thì lặng lẽ ở lại bên trong VPC. Nếu tắt tùy chọn này, endpoint vẫn tồn tại nhưng không có gì đi qua nó.

Chỉ chọn **duy nhất private subnet**. Endpoint được tính phí theo từng subnet mà nó được đặt vào, nên chọn cả ba subnet sẽ làm chi phí tăng gấp ba mà chẳng được lợi gì — không có gì trong public subnet gọi API của AWS cả.

#### Tạo gateway endpoint cho S3

Endpoint S3 được cấu hình theo cách khác:

| Mục | Giá trị |
|---|---|
| Service name | `com.amazonaws.ap-southeast-1.s3` |
| Type | **Gateway** |
| VPC | VPC đã tạo ở 5.3.1 |
| Route tables | chọn route table **private** |
| Policy | Full access |

Ở đây không có subnet hay security group để chọn. Việc chọn route table private sẽ thêm một route cho prefix list của S3, và đó chính là thứ đẩy lưu lượng S3 đi qua endpoint.

#### Kiểm chứng

Cả chín endpoint đều phải ở trạng thái `available`:

![danh sách endpoint](/images/5-Workshop/5.3-Network/ep-list.png)

Kiểm tra để chắc chắn mọi interface endpoint đều báo `PrivateDnsEnabled: true`. Một endpoint ở trạng thái `available` nhưng tắt Private DNS sẽ không mang bất kỳ lưu lượng nào, và lỗi sinh ra trông hệt như một timeout thông thường.

#### Về chi phí

Thiết kế này thường được giới thiệu như một phương án rẻ hơn NAT Gateway. Với số lượng endpoint như thế này thì điều đó không hẳn đúng, và cũng nên nói thẳng ra.

Một interface endpoint tính phí khoảng 0.01–0.013 USD mỗi giờ cho mỗi Availability Zone mà nó được đặt vào, cộng thêm một khoản phí xử lý dữ liệu nhỏ theo gigabyte. Tám interface endpoint trong một AZ do đó rơi vào khoảng 60–80 USD mỗi tháng. Trong khi một NAT Gateway đơn lẻ tốn khoảng 0.06 USD mỗi giờ, tức chừng 43 USD mỗi tháng, cộng phí theo gigabyte của riêng nó.

Vậy nếu chỉ xét giá, một NAT Gateway rẻ hơn tám interface endpoint. Cái mà endpoint đem lại là private subnet thực sự không có bất kỳ đường nào ra Internet — lưu lượng tới dịch vụ AWS không bao giờ đi qua mạng công cộng, và một container bị chiếm quyền cũng không thể gọi ra một máy chủ tùy ý bên ngoài. Gateway endpoint cho S3 lại miễn phí, mà S3 mới là nơi phần lớn dung lượng dữ liệu đi qua, nên phí xử lý dữ liệu vẫn ở mức thấp.

{{% notice note %}}
Nếu trong hệ thống của bạn chi phí quan trọng hơn việc cô lập đường ra Internet, hãy bỏ bớt những endpoint không cần thiết. `ssmmessages` và `ec2messages` chỉ tồn tại để phục vụ ECS Exec; nếu bạn không có ý định mở shell vào task đang chạy thì bỏ hai cái đó đi là tiết kiệm được hai endpoint. Hãy kiểm tra giá hiện hành tại [trang giá VPC](https://aws.amazon.com/vpc/pricing/) — các con số ở trên chỉ là ước lượng và áp dụng riêng cho `ap-southeast-1`.
{{% /notice %}}

#### Bước tiếp theo

Phần mạng đã hoàn tất: một VPC với public và private subnet, định tuyến chính xác, chín security group mô tả rõ ai được nói chuyện với ai, và chín endpoint cho phép private subnet truy cập dịch vụ AWS mà không cần đường ra Internet. Mục 5.4 sẽ đặt tầng dữ liệu — RDS, ElastiCache, Amazon MQ và EFS — vào bên trong nền tảng mạng này.
