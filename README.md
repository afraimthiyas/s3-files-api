# S3 Files API (NestJS + AWS IAM)

JWT-authenticated file API. Files go straight to S3 via presigned URLs; metadata lives in DynamoDB.
No AWS keys anywhere: the app uses its IAM role.

## 1. Create AWS resources (us-east-1 example)
```bash
aws s3api create-bucket --bucket my-app-bucket --region us-east-1
aws s3api put-public-access-block --bucket my-app-bucket \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws dynamodb create-table --table-name app-users --billing-mode PAY_PER_REQUEST \
  --attribute-definitions AttributeName=email,AttributeType=S --key-schema AttributeName=email,KeyType=HASH
aws dynamodb create-table --table-name app-files --billing-mode PAY_PER_REQUEST \
  --attribute-definitions AttributeName=userId,AttributeType=S AttributeName=fileId,AttributeType=S \
  --key-schema AttributeName=userId,KeyType=HASH AttributeName=fileId,KeyType=RANGE
```
Browser uploads need a bucket CORS rule allowing PUT from your frontend origin.

## 2. IAM role
Edit `ACCOUNT_ID`/bucket in `iam/task-role-policy.json`, then:
```bash
aws iam create-role --role-name s3-files-api-task --assume-role-policy-document file://iam/trust-policy.json
aws iam put-role-policy --role-name s3-files-api-task --policy-name app --policy-document file://iam/task-role-policy.json
```
Use this as the **task role** of your ECS/Fargate service (or swap the trust policy for EC2/Lambda).

## 3. Run locally
```bash
cp .env.example .env      # edit values
aws sso login             # or any profile; set AWS_PROFILE
npm install && npm run start:dev
```

## 4. API
| Method | Path | Body |
|---|---|---|
| POST | /auth/register, /auth/login | `{email, password}` -> `{accessToken}` |
| POST | /files/upload-url | `{filename, contentType}` -> `{fileId, uploadUrl}`; then HTTP PUT the file to `uploadUrl` with the same Content-Type |
| GET | /files | list yours |
| GET | /files/:id/download-url | presigned GET |
| DELETE | /files/:id | delete |

Send `Authorization: Bearer <token>`.

## 5. Deploy on ECS Fargate
```bash
aws ecr create-repository --repository-name s3-files-api
docker build -t s3-files-api . && docker tag s3-files-api ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/s3-files-api
aws ecr get-login-password | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/s3-files-api
```
Create a Fargate service behind an ALB (health check `/health`, port 3000). Set env vars from `.env.example`,
and inject `JWT_SECRET` from Secrets Manager via the task definition `secrets` field
(the *execution* role needs `secretsmanager:GetSecretValue`).

---

# Part 2: Event-driven AI processing + CI/CD

```
Client -> presigned PUT -> S3 --(ObjectCreated)--> SQS --> Lambda --> Bedrock (LLM)
                                                            \--> DynamoDB (aiSummary, aiTags, aiStatus)
```
`GET /files/:id` returns `aiStatus` (`pending | done | skipped | failed`), `aiSummary`, `aiTags`.
Only text-like files (text/*, JSON) are analyzed; others are marked `skipped`.

## Create the pipeline
```bash
# Queue + dead-letter queue
aws sqs create-queue --queue-name file-uploaded-dlq
aws sqs create-queue --queue-name file-uploaded \
  --attributes '{"VisibilityTimeout":"120","RedrivePolicy":"{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:ACCOUNT_ID:file-uploaded-dlq\",\"maxReceiveCount\":\"3\"}"}'
# Allow S3 to send to the queue (set the Policy attribute on file-uploaded: Principal s3.amazonaws.com,
# Action sqs:SendMessage, Condition aws:SourceArn = arn:aws:s3:::my-app-bucket)

# S3 -> SQS notification for uploads/
aws s3api put-bucket-notification-configuration --bucket my-app-bucket --notification-configuration \
 '{"QueueConfigurations":[{"QueueArn":"arn:aws:sqs:us-east-1:ACCOUNT_ID:file-uploaded","Events":["s3:ObjectCreated:*"],"Filter":{"Key":{"FilterRules":[{"Name":"prefix","Value":"uploads/"}]}}}]}'

# Worker role (trust: lambda.amazonaws.com, policy: iam/worker-role-policy.json), then:
cd worker && npm install && npm run build && cd dist && zip ../worker.zip index.js && cd ..
aws lambda create-function --function-name files-ai-worker --runtime nodejs20.x --handler index.handler \
  --role arn:aws:iam::ACCOUNT_ID:role/files-ai-worker-role --timeout 60 --memory-size 256 \
  --zip-file fileb://worker.zip \
  --environment 'Variables={FILES_TABLE=app-files,BEDROCK_MODEL_ID=<your-bedrock-model-id>}'
aws lambda create-event-source-mapping --function-name files-ai-worker \
  --event-source-arn arn:aws:sqs:us-east-1:ACCOUNT_ID:file-uploaded --batch-size 5
```
Enable access to your chosen model in the Bedrock console and use its model/inference-profile ID for `BEDROCK_MODEL_ID`.

## GitHub Actions
`.github/workflows/ci-cd.yml` builds and type-checks the API and worker on every PR; on `main` it pushes the
image to ECR, redeploys ECS, and updates the Lambda. Setup: create an IAM role that trusts GitHub's OIDC provider
for your repo (permissions: ECR push, `ecs:UpdateService`, `lambda:UpdateFunctionCode`) and save its ARN as the
repository secret `AWS_DEPLOY_ROLE_ARN`. No long-lived AWS keys are stored.
