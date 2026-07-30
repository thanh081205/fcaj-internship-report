---
title : "CI/CD với GitHub Actions"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

Mọi thứ từ đầu tới giờ đều làm bằng tay. Mục này thay thế bằng một pipeline: push lên `main`, và GitHub Actions sẽ build cả hai image, chạy migration database, rồi cuộn các ECS service sang phiên bản mới — mà không lưu bất kỳ AWS access key nào ở đâu cả.

#### Vì sao dùng OIDC thay vì access key

Cách hiển nhiên để GitHub Actions deploy được là tạo một IAM user, sinh access key, rồi dán vào repository secret. Khóa đó tồn tại vĩnh viễn, dùng được từ bất kỳ đâu trên Internet, và có hiệu lực cho tới khi ai đó nhớ ra phải xoay vòng nó. Nếu nó rò rỉ — qua một dòng log, một bản fork, một lần chia sẻ màn hình — thì cả tài khoản bị phơi ra.

**OIDC federation** loại bỏ hoàn toàn cái khóa đó. GitHub ký một token ngắn hạn mô tả lần chạy workflow, AWS xác minh chữ ký đó với khóa công khai của GitHub, rồi cấp thông tin đăng nhập tạm thời chỉ có hiệu lực trong thời gian job chạy. Không có gì tồn tại lâu dài được lưu ở cả hai phía.

#### Đăng ký GitHub làm identity provider

**IAM** → **Identity providers** → **Add provider**:

| Mục | Giá trị |
|---|---|
| Provider type | OpenID Connect |
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

![oidc provider](/images/5-Workshop/5.7-CICD/provider.png)

Việc này chỉ làm một lần cho mỗi tài khoản AWS. Nếu provider đã tồn tại thì dùng lại.

#### Tạo role dùng để deploy

Identity provider mới chỉ nói với AWS rằng nó tin chữ ký của GitHub. Nó chưa cấp quyền gì cả. Việc đó thuộc về một role mà trust policy của nó chỉ rõ *repository nào* được phép nhận vai, và sau đó *được làm những gì*.

**IAM** → **Roles** → **Create role** → **Custom trust policy**, rồi dán vào:

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::{id}:oidc-provider/token.actions.githubusercontent.com"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
					"token.actions.githubusercontent.com:sub": "repo:{github-owner}/{repo_name}:ref:refs/heads/main"
				}
			}
		}
	]
}
```

Ở trang permissions, đính kèm **chính sách quyền deploy bạn đã tạo ở mục 5.2** — chính sách có các `Sid` là `ECRAuthToken`, `ECRRepositoryOperations`, `ECSServiceManagement` và `RestrictedPassRoleToECSOnly`. Chính sách đó đã được giới hạn đúng bằng những gì pipeline cần làm: đẩy image lên ECR, đăng ký và cập nhật task definition, chạy task migration, và truyền `ecsTaskExecutionRole` cho ECS chứ không truyền gì khác.

Đặt tên role là **`GitHubActionsDeployRole`** — tên này phải khớp với biến `AWS_ROLE_ARN` ở bước kế tiếp.

{{% notice warning %}}
Điều kiện `sub` chính là toàn bộ ranh giới bảo mật của thiết lập này. Bỏ nó đi, hoặc thay bằng `"token.actions.githubusercontent.com:sub": "*"`, thì **bất kỳ** workflow GitHub Actions nào trong **bất kỳ** repository nào trên GitHub cũng có thể nhận vai này và deploy vào tài khoản của bạn. Riêng điều kiện `aud` không giới hạn được gì cả — mọi token OIDC của GitHub đều mang audience là `sts.amazonaws.com`.
{{% /notice %}}

Giá trị `sub` ở trên giới hạn việc nhận vai vào đúng nhánh `main` của một repository. Hai biến thể thường gặp:

| Mục tiêu | Giá trị `sub` | Toán tử điều kiện |
|---|---|---|
| Chỉ nhánh `main` (khuyến nghị) | `repo:{github-owner}/{repo_name}:ref:refs/heads/main` | `StringEquals` |
| Mọi nhánh hoặc tag trong repo đó | `repo:{github-owner}/{repo_name}:*` | `StringLike` |

Chỉ dùng `StringLike` khi giá trị có chứa ký tự đại diện. Ký tự đại diện nằm trong `StringEquals` sẽ bị so khớp theo nghĩa đen, tạo ra một role âm thầm không bao giờ nhận vai được — lỗi hiện ra trông như lỗi phân quyền chứ không giống lỗi gõ nhầm.

#### Thêm repository variable

Trên GitHub: **Settings** → **Secrets and variables** → **Actions** → **Variables**:

| Name | Value |
|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::<account-id>:role/GitHubActionsDeployRole` |
| `AWS_REGION` | `ap-southeast-1` |
| `ECS_CLUSTER` | `vsp-ecs-cluster` |

Đây là **variable**, không phải secret — không giá trị nào trong số đó là bí mật, và variable hiện ra trong log giúp việc gỡ lỗi dễ hơn nhiều. Không có secret nào cần thêm cả: đó chính là điểm mấu chốt của OIDC.

#### Workflow này làm gì, và vì sao theo thứ tự đó

**Gắn tag theo commit SHA.** `${GITHUB_SHA::7}` cho mỗi bản build một tag duy nhất, truy vết được. Bạn nhìn vào một task đang chạy là biết chính xác nó sinh ra từ commit nào, và rollback nghĩa là deploy lại một tag đã biết thay vì build lại từ đầu.

**Migrate trước, deploy sau.** Job migration chặn các job deploy thông qua `needs`. Nếu migration thất bại, không có mã mới nào lên tới cluster — phiên bản cũ vẫn chạy với schema cũ, tức là một hệ thống đang hoạt động. Nếu deploy trước rồi mới migrate, ta sẽ có mã mới chạy trên schema mà nó không hiểu.

**Thất bại khi exit code khác 0.** `aws ecs run-task` trả về thành công ngay khi task được *lên lịch*, chứ không phải khi nó chạy xong đúng. Nếu không có bước `describe-tasks` kiểm tra `exitCode` một cách tường minh, một migration thất bại vẫn sẽ được báo là bước màu xanh.

**Render thay vì viết lại task definition.** Action render tải task definition đang chạy về và chỉ thay đúng trường image, nhờ vậy secret, port mapping và giới hạn tài nguyên được giữ nguyên chính xác. Tự viết tay JSON trong pipeline sẽ khiến nó lệch khỏi thực tế ngay khi có người đổi thứ gì đó trong console.

**Chờ tới khi ổn định.** `wait-for-service-stability: true` kết hợp với circuit breaker đã bật ở mục 5.5 nghĩa là một image hỏng sẽ vừa làm workflow thất bại, vừa khiến service tự động rollback.

![lần chạy workflow](/images/5-Workshop/5.7-CICD/run.png)

#### Thử nghiệm

```
git commit --allow-empty -m "test: trigger deployment pipeline"
git push origin main
```

Theo dõi lần chạy ở tab **Actions**. Khi xong, xác nhận task đang chạy dùng đúng tag mới:

```
aws ecs describe-services --cluster vsp-ecs-cluster --services api-service \
  --query "services[0].taskDefinition" --output text

aws ecs describe-task-definition --task-definition vsp-api-service \
  --query "taskDefinition.containerDefinitions[0].image" --output text
```

Tag của image phải khớp với commit SHA rút gọn mà bạn vừa push.


