# AWS IAM Role và `iam:PassRole`

Status: VERIFIED
Last reviewed: 2026-07-14

## Vấn đề

Trong AWS, **IAM Role** và **`iam:PassRole`** thường xuất hiện cùng nhau nên dễ bị hiểu nhầm là một khái niệm. “RolePass” hoặc “rolepass” không phải tên chính thức của một AWS resource; thuật ngữ đúng là action/permission **`iam:PassRole`**.

Ghi chú này trả lời:

- IAM Role là gì và ai có thể assume role?
- Trust policy khác permissions policy thế nào?
- `sts:AssumeRole` khác `iam:PassRole` ở đâu?
- Vì sao tạo SageMaker training job, Lambda function hoặc EC2 instance thường cần `iam:PassRole`?
- Làm sao tránh privilege escalation khi cấp quyền pass role?

## Ghi chú

### 1. Mental model ngắn gọn

```text
IAM Role      = một bộ danh tính + quyền có thể được assume tạm thời
AssumeRole    = principal tự chuyển sang dùng quyền của role
PassRole      = principal giao role cho một AWS service sử dụng
```

Ví dụ SageMaker:

```text
Developer / CI role
   │
   ├── sagemaker:CreateTrainingJob
   └── iam:PassRole trên SageMakerExecutionRole
                 │
                 ▼
       SageMaker service assume role
                 │
                 ├── đọc training data từ S3
                 ├── pull image từ ECR
                 ├── ghi model artifact vào S3
                 └── ghi logs/metrics
```

Developer không trực tiếp assume execution role trong flow này. Developer **pass** role cho SageMaker; SageMaker mới là principal **assume** role đó.

### 2. IAM Role là gì?

IAM Role là một IAM identity có permissions policies, tương tự IAM User. Điểm khác biệt quan trọng:

- Role không gắn cố định với một người.
- Role được thiết kế để principal phù hợp có thể assume.
- Role không có password hoặc access key dài hạn.
- Khi được assume, AWS STS cấp temporary security credentials cho role session.

Role có thể được assume bởi:

- IAM user hoặc IAM role khác.
- Principal trong cùng account hoặc account khác.
- AWS service như EC2, Lambda hoặc SageMaker.
- Federated identity từ SAML/OIDC.

Các use case phổ biến:

- Workload chạy trên AWS cần gọi API AWS.
- Cross-account access.
- Người dùng nhận quyền tạm thời qua federation/Identity Center.
- AWS service thực hiện tác vụ thay cho khách hàng.

### 3. Hai “mặt” của một role

Để hiểu role, luôn kiểm tra hai policy độc lập:

#### Trust policy — Ai được assume role?

Trust policy là resource-based policy bắt buộc gắn với role. Nó xác định trusted principal và thường cho phép `sts:AssumeRole`.

Ví dụ cho SageMaker:

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {
      "Service": "sagemaker.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }
}
```

Câu hỏi mà policy này trả lời:

> SageMaker có được phép trở thành role này không?

#### Permissions policy — Sau khi assume, role được làm gì?

Permissions policy là identity-based policy gắn với role. Nó mô tả actions và resources mà role session được sử dụng.

Ví dụ tối giản:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::ml-training-data/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::ml-model-artifacts/*"
    }
  ]
}
```

Câu hỏi mà policy này trả lời:

> Sau khi SageMaker assume role, nó được thao tác lên resource nào?

Một trust policy rộng không tự cấp quyền S3. Ngược lại, permissions policy rất mạnh cũng chưa dùng được nếu không principal nào được trust để assume role.

### 4. `sts:AssumeRole` là gì?

`sts:AssumeRole` là API/action của AWS Security Token Service để tạo một role session và trả về temporary credentials.

Flow khái quát:

```text
Caller → sts:AssumeRole → AWS STS → temporary credentials
                                      │
                                      ▼
                               Role session
```

Để assume role thành công, thông thường cần đồng thời:

1. Trust policy của target role tin cậy caller.
2. Caller được phép gọi `sts:AssumeRole` nếu policy evaluation của tình huống đó yêu cầu.
3. Không có explicit deny từ SCP, permissions boundary, session policy hoặc policy áp dụng khác.

Sau khi assume, caller dùng temporary credentials để gọi API dưới danh tính role session.

### 5. `iam:PassRole` là gì?

`iam:PassRole` là một IAM permission cho phép principal **giao một IAM role cho AWS service** khi tạo hoặc cấu hình resource/job.

`iam:PassRole`:

- Không phải một API độc lập.
- Không tạo temporary credentials cho caller.
- Không khiến caller trực tiếp trở thành role.
- Được kiểm tra khi request tới service chứa role ARN.
- Chỉ pass role trực tiếp cho service trong cùng AWS account.

Ví dụ request tạo SageMaker training job có `RoleArn`. AWS kiểm tra caller có cả:

- `sagemaker:CreateTrainingJob`.
- `iam:PassRole` trên đúng execution role được truyền vào.

Sau đó SageMaker dùng trust relationship để assume execution role khi chạy job.

### 6. `AssumeRole` và `PassRole` khác nhau thế nào?

| Câu hỏi | `sts:AssumeRole` | `iam:PassRole` |
|---|---|---|
| Ai dùng role? | Caller | AWS service |
| Caller nhận temporary credentials? | Có | Không |
| Là API call riêng? | Có | Không, là permission được service kiểm tra |
| Policy đặt ở đâu? | Trust policy của target role và policy liên quan của caller | Identity-based policy của caller |
| Mục đích | Chuyển sang role session | Gắn/giao role cho service hoặc resource |
| CloudTrail | Có event AssumeRole | Không có event PassRole riêng |

Mẹo nhớ:

```text
Assume = “Tôi sử dụng role này.”
Pass   = “Service này sẽ sử dụng role này.”
```

### 7. Ba điều kiện để pass role hoạt động

Trong flow phổ biến, cần đủ ba mảnh:

1. Caller có permission tạo/cập nhật resource, ví dụ `lambda:CreateFunction` hoặc `sagemaker:CreateTrainingJob`.
2. Caller có `iam:PassRole` trên role ARN được chọn.
3. Trust policy của role cho phép service đích assume role.

```text
Caller policy                  Role trust policy
─────────────                  ─────────────────
CreateTrainingJob  ─────┐
iam:PassRole       ─────┼──► SageMaker resource/job
                       └──► sagemaker.amazonaws.com can AssumeRole
```

Thiếu (1): caller không được tạo resource.

Thiếu (2): thường nhận lỗi “not authorized to perform iam:PassRole”.

Thiếu (3): resource có thể không được tạo hoặc service không thể sử dụng role đúng cách, tùy API/service.

### 8. Ví dụ least-privilege PassRole cho SageMaker

Policy sau cho phép caller chỉ pass các role có naming convention xác định và chỉ tới SageMaker:

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Sid": "PassApprovedRolesToSageMakerOnly",
    "Effect": "Allow",
    "Action": "iam:PassRole",
    "Resource": "arn:aws:iam::111122223333:role/mlops/sagemaker-*",
    "Condition": {
      "StringEquals": {
        "iam:PassedToService": "sagemaker.amazonaws.com"
      }
    }
  }
}
```

Các lớp giới hạn:

- `Resource` giới hạn role nào được pass.
- Path/name giới hạn một nhóm role được quản trị riêng.
- `iam:PassedToService` giới hạn service nhận role.

AWS cảnh báo không nên dựa vào `iam:ResourceTag`/`aws:ResourceTag` để kiểm soát `iam:PassRole`, vì cách này không cho kết quả đáng tin cậy. Nên scope trực tiếp bằng role ARN/path và condition key được hỗ trợ.

### 9. Vì sao `iam:PassRole` nhạy cảm?

Rủi ro lớn nhất là **privilege escalation gián tiếp**.

Ví dụ:

1. Alice không có quyền đọc bucket bí mật.
2. Alice có quyền tạo Lambda function.
3. Alice có `iam:PassRole` với `Resource: "*"`.
4. Trong account tồn tại AdminRole hoặc role đọc bucket bí mật và trust Lambda.
5. Alice tạo function dùng role mạnh đó.
6. Lambda đọc dữ liệu thay Alice.

Alice không assume AdminRole trực tiếp, nhưng đã điều khiển một service dùng role mạnh hơn quyền của mình.

Một policy nguy hiểm:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "*"
}
```

Mức độ nguy hiểm phụ thuộc vào các quyền tạo/cập nhật workload mà principal đang có. `PassRole` thường phải được phân tích cùng `lambda:CreateFunction`, `ec2:RunInstances`, `ecs:RunTask`, `sagemaker:CreateTrainingJob`, `glue:CreateJob` và các action tương tự.

### 10. Security checklist

- Scope `iam:PassRole` tới ARN/path cụ thể; tránh `Resource: "*"`.
- Dùng `iam:PassedToService` khi service hỗ trợ.
- Tách role theo workload, môi trường và mức độ nhạy cảm.
- Không cho người có quyền PassRole tự ý sửa permissions hoặc trust policy của role được pass.
- Dùng permissions boundary để giới hạn quyền tối đa của role do team tự tạo.
- Dùng SCP để đặt guardrail ở cấp AWS Organization khi phù hợp.
- Kiểm tra cả hai phía: caller policy và target role trust/permissions policies.
- Với service principal truy cập resource, dùng `aws:SourceArn`, `aws:SourceAccount`, `aws:SourceOrgID` hoặc `aws:SourceOrgPaths` nếu service hỗ trợ để giảm confused deputy risk.
- Dùng IAM Access Analyzer và policy validation để phát hiện quyền rộng hoặc external access.
- Theo dõi CloudTrail event tạo/cập nhật resource có chứa role ARN.

### 11. CloudTrail và điều tra PassRole

Vì `iam:PassRole` không phải API call nên CloudTrail không tạo event tên `PassRole`.

Muốn điều tra ai đã pass role nào:

1. Tìm event tạo hoặc cập nhật resource, ví dụ `CreateTrainingJob`, `CreateFunction` hoặc `RunInstances`.
2. Xem `userIdentity` để biết caller.
3. Xem request parameters để tìm role ARN.
4. Đối chiếu target role, service và thời điểm thực hiện.

Đây là điểm dễ bỏ sót khi viết detection rule: tìm `eventName = PassRole` sẽ không có kết quả.

### 12. Các loại service role dễ nhầm

#### Service role

Role do khách hàng quản lý, được AWS service assume để thực hiện hành động thay khách hàng. Ví dụ SageMaker execution role.

#### Service-linked role

Một loại service role liên kết chặt với một AWS service, do service sở hữu. IAM admin có thể xem nhưng không chỉnh sửa permissions của service-linked role theo cách thông thường. Một số service tự tạo loại role này.

#### Execution role

Tên mang tính use case cho role mà workload/service assume lúc thực thi, ví dụ Lambda execution role hoặc SageMaker execution role. Về bản chất nó vẫn là IAM Role.

### 13. SageMaker MLOps example

Một pipeline tạo training job có thể có hai role:

```text
PipelineRole / CI role
├── sagemaker:CreateTrainingJob
└── iam:PassRole → SageMakerTrainingExecutionRole

SageMakerTrainingExecutionRole
├── trust: sagemaker.amazonaws.com
├── s3:GetObject → training data
├── s3:PutObject → model artifacts
├── ecr:* cần thiết → training image
└── logs:* cần thiết → observability
```

Hai role phục vụ hai trách nhiệm khác nhau:

- **Pipeline/CI role:** được phép yêu cầu SageMaker tạo job và chỉ định execution role.
- **Execution role:** là quyền thực tế của training container khi job chạy.

Việc tách này giúp người vận hành pipeline không cần trực tiếp có mọi quyền S3/ECR mà workload cần.

### 14. Các lỗi phổ biến

#### “Not authorized to perform iam:PassRole”

Kiểm tra:

- Caller có Allow `iam:PassRole` không?
- `Resource` có khớp chính xác role ARN/path không?
- `iam:PassedToService` có đúng service principal không?
- Có explicit deny từ SCP, boundary hoặc session policy không?

#### Có PassRole nhưng service vẫn không chạy được

Kiểm tra:

- Trust policy có đúng service principal không?
- Execution role có đủ quyền trên S3, ECR, KMS, CloudWatch Logs… không?
- Resource policy như bucket policy hoặc KMS key policy có cho phép không?
- Network/VPC configuration có chặn truy cập không?

#### Nhầm rằng PassRole cho phép assume role

`iam:PassRole` không cấp temporary credentials cho caller. Nếu caller cần tự dùng role, flow phù hợp là `sts:AssumeRole` cùng trust relationship tương ứng.

## Điều rút ra

- IAM Role là identity có quyền nhưng không có long-term credentials; principal phù hợp assume role để nhận temporary credentials.
- Trust policy trả lời **ai được assume**; permissions policy trả lời **role được làm gì**.
- `sts:AssumeRole` dành cho principal tự dùng role; `iam:PassRole` dành cho việc giao role cho AWS service.
- `iam:PassRole` không phải API call và không có CloudTrail event riêng.
- Một workload thường cần cả quyền tạo resource, quyền pass đúng role và trust policy cho service đích.
- `iam:PassRole` quá rộng có thể dẫn tới privilege escalation, đặc biệt khi đi cùng quyền tạo Lambda, EC2, ECS, Glue hoặc SageMaker workloads.
- Least privilege tốt nhất là giới hạn role ARN/path, service đích và quyền thực tế của execution role.

## Nguồn

- [AWS IAM — IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [AWS IAM — Grant a user permissions to pass a role to an AWS service](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html)
- [AWS IAM — Policies and permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [AWS IAM — IAM and AWS STS condition context keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_iam-condition-keys.html)
- [AWS IAM — The confused deputy problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)
- [Amazon SageMaker AI — How to use execution roles](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html)
- [Amazon SageMaker API — CreateTrainingJob](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateTrainingJob.html)
