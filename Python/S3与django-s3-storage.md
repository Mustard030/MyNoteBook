## **ARN 的通用格式**

`arn:partition:service:region:account-id:resource`

- `arn`：固定前缀
- `partition`：AWS 分区，例如 `aws`（标准 AWS 区域）、`aws-cn`（中国）、`aws-us-gov`（政府云）
- `service`：资源所属服务，例如 `s3`, `iam`, `lambda`
- `region`：区域，例如 `us-east-1`（有些服务忽略）
- `account-id`：AWS 账号 ID（有些服务忽略）
- `resource`：具体资源，可以是名称、路径或 ID