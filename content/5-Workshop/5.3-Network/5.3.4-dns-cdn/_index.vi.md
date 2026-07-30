---
title : "DNS và tầng edge CloudFront"
date : 2024-01-01
weight : 4
chapter : false
pre : " <b> 5.3.4 </b> "
---

VPC lo phần xử lý lưu lượng sau khi nó đã đến nơi. Trang này thiết lập phần *làm sao để nó đến được*: public DNS zone, chứng chỉ TLS, và distribution CloudFront đứng trước toàn bộ hệ thống.

Các bước này được đưa lên sớm vì một lý do rất thực tế. Chứng chỉ ACM chỉ được cấp sau khi việc xác thực qua DNS thành công, mà lan truyền DNS lại đúng là thứ duy nhất trong workshop này bạn không thể ép nhanh hơn — việc trỏ nameserver về Route 53 có thể mất từ vài phút đến vài giờ.

#### Tạo public hosted zone

Mở console **Route 53** → **Hosted zones** → **Create hosted zone**.

| Mục | Giá trị |
|---|---|
| Domain name | tên miền của bạn, ví dụ `example.com` |
| Type | Public hosted zone |

Sau khi tạo, zone sẽ có sẵn một bản ghi `NS` liệt kê bốn nameserver của Route 53, cùng một bản ghi `SOA`. Chính bốn nameserver đó là thứ khiến zone này có thẩm quyền với tên miền.

![hosted zone](/images/5-Workshop/5.3-Network/hosted-zone.png)

Nếu bạn đăng ký tên miền **qua Route 53**, việc ủy quyền đã xong sẵn và có thể bỏ qua bước tiếp theo. Nếu bạn đăng ký ở **nhà cung cấp khác**, hãy vào trang quản trị của họ và thay bốn nameserver mặc định bằng bốn nameserver của zone này. Không phần nào còn lại của workshop hoạt động được cho tới khi thay đổi đó có hiệu lực.

#### Yêu cầu hai chứng chỉ TLS

Đây là chỗ nhiều người vấp: bạn cần **hai chứng chỉ cho cùng một tên miền**, ở hai region khác nhau.

| Chứng chỉ | Region | Dùng cho |
|---|---|---|
| `example.com` + `*.example.com` | **`us-east-1`** | CloudFront |
| `example.com` + `*.example.com` | `ap-southeast-1` | Application Load Balancer |

{{% notice warning %}}
CloudFront chỉ chấp nhận chứng chỉ được cấp ở **US East (N. Virginia) — `us-east-1`**. Đây không phải một tùy chọn hay giá trị mặc định có thể đổi, mà là ràng buộc cứng, vì CloudFront là dịch vụ toàn cầu với control plane đặt tại region đó. Một chứng chỉ cấp ở `ap-southeast-1` sẽ đơn giản là không xuất hiện trong danh sách chọn chứng chỉ của CloudFront, mà không kèm bất kỳ lời giải thích nào.
{{% /notice %}}

Với từng chứng chỉ: mở **AWS Certificate Manager**, kiểm tra lại bộ chọn region ở góc trên bên phải cho đúng, rồi chọn **Request** → **Request a public certificate**.

| Mục | Giá trị |
|---|---|
| Fully qualified domain name | `example.com` |
| Additional name | `*.example.com` |
| Validation method | **DNS validation** |
| Key algorithm | RSA 2048 |

Thêm wildcard `*.example.com` bên cạnh tên miền gốc nghĩa là một chứng chỉ duy nhất bao phủ được `app.example.com`, `api.example.com` và bất kỳ tên nào bạn thêm về sau, nên không bao giờ phải lặp lại bước này.

Hãy chọn **DNS validation** thay vì xác thực qua email. DNS validation tự động gia hạn chừng nào bản ghi xác thực còn nằm trong zone; còn xác thực qua email thì lần nào cũng cần một người thật vào bấm link.

#### Xác thực chứng chỉ

Mỗi chứng chỉ khi tạo ra đều ở trạng thái `Pending validation` kèm một bản ghi `CNAME` mà nó cần nhìn thấy trong DNS của bạn. Vì hosted zone nằm cùng tài khoản, ACM có thể tự ghi bản ghi đó: mở chứng chỉ và chọn **Create records in Route 53**.

![chứng chỉ 1 đã cấp](/images/5-Workshop/5.3-Network/cert1.png)

![chứng chỉ 2 đã cấp](/images/5-Workshop/5.3-Network/cert2.png)

#### Thiết kế tầng edge

CloudFront là thứ duy nhất mà người dùng thực sự kết nối tới. Application Load Balancer tuy có tên miền công khai nhưng không người dùng nào được trỏ thẳng tới đó — mọi request đều đi vào qua CloudFront, và CloudFront chuyển tiếp tới ALB hay tới S3 tùy theo đường dẫn. Sử dụng AWS WAF để để bảo vệ cụm server.

Tạo clodfront gồm distribution có **hai origin**:

| Origin | Trỏ tới | Phục vụ |
|---|---|---|
| ALB origin | tên miền của ALB | Request GraphQL, REST và tìm kiếm |
| S3 origin | bucket chứa video | File video và các segment DASH |

![cloudfront origins](/images/5-Workshop/5.3-Network/cloudfront-origin.png)

và **ba behavior**, xét theo thứ tự:

| Thứ tự | Path pattern | Origin | Cache policy | Lý do |
|---|---|---|---|---|
| 0 | `/public/*` | S3 | CachingOptimized | Segment video công khai — rất đáng cache |
| 1 | `/private/*` | S3 | CachingOptimized | Media có kiểm soát truy cập |
| 2 | `Default (*)` | ALB | CachingDisabled | Lưu lượng API — tuyệt đối không được cache |

![cloudfront behaviors](/images/5-Workshop/5.3-Network/cloudfront-behaviors.png)

Cấu hình đúng behavior mặc định quan trọng hơn vẻ ngoài của nó rất nhiều. Nếu cache policy mặc định có cache lại response, CloudFront sẽ vô tư trả response GraphQL của người dùng này cho người dùng khác, bởi header `Authorization` không nằm trong cache key trừ khi bạn chủ động thêm vào. Hãy đặt `CachingDisabled` cho behavior mặc định và chuyển tiếp toàn bộ header, cookie và query string về origin.

Lý do đặt CloudFront trước ALB ngay từ đầu:

+ **Phân phối video chiếm phần lớn lưu lượng.** Các segment DASH là file tĩnh được yêu cầu rất nhiều lần. Phục vụ chúng từ edge location thay vì từ `ap-southeast-1` vừa nhanh hơn cho người xem, vừa rẻ hơn so với việc trả phí egress của ALB và S3 cho từng request.
+ **TLS kết thúc ngay tại edge**, gần người dùng, giúp giảm số vòng bắt tay ở lần kết nối đầu tiên.
+ **ALB được che chắn.** Chỉ dải IP của CloudFront mới cần truy cập tới nó, và sau này có thể gắn AWS WAF ở tầng edge mà không phải đụng vào ứng dụng.

#### Bản ghi DNS sẽ thêm sau

Khi distribution được tạo ở mục 5.6, zone sẽ có thêm một bản ghi nữa:

| Name | Type | Value |
|---|---|---|
| `app.example.com` | A — Alias | distribution CloudFront |

Hãy dùng bản ghi **alias**, đừng dùng `CNAME`. Alias là tính năng riêng của Route 53, phân giải tới địa chỉ IP hiện hành của tài nguyên AWS, không tính phí truy vấn, và — khác với `CNAME` — được phép đặt ngay tại gốc zone nếu sau này bạn muốn chính `example.com` trỏ tới CloudFront.

#### Một zone thứ hai, dạng private

Route 53 còn đảm nhận cả việc phân giải tên *bên trong* VPC. Ở mục 5.5, các ECS service sẽ đăng ký vào một private hosted zone tên `vsp.internal` thông qua AWS Cloud Map. Nhờ đó mỗi task có một tên nội bộ ổn định như `api-service.vsp.internal`, để hai service tìm thấy nhau qua gRPC mà không phải ghi cứng những địa chỉ IP vốn thay đổi sau mỗi lần deploy.

Zone đó được tạo tự động khi thiết lập Cloud Map namespace nên ở đây không cần làm gì, nhưng cũng nên biết rằng hai zone tồn tại vì hai mục đích hoàn toàn khác nhau: public zone dẫn người dùng tới tầng edge, còn private zone giúp các container tìm thấy nhau.

![private zone](/images/5-Workshop/5.3-Network/vsp-internal-zone.png)