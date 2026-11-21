# Guided Lab: Serverless Notes API with Lambda, API Gateway HTTP API and DynamoDB

## Objective

Build a small serverless REST-style API (`POST /notes`, `GET /notes/{id}`) backed by DynamoDB. Give the Lambda function a least-privilege IAM role, turn on access logging, and observe throttling.

## Scenario

A startup needs a tiny notes API that costs nothing when idle. You use an API Gateway HTTP API in front of one Lambda function (Python 3.12), which reads and writes a DynamoDB on-demand table. Nothing runs when nobody calls it.

## Services Used

- Amazon API Gateway (HTTP API)
- AWS Lambda
- Amazon DynamoDB (on-demand)
- AWS IAM
- Amazon CloudWatch Logs

## Prerequisites

- An AWS account with an admin-capable IAM user or role, and AWS CLI v2 configured for `us-east-1`.
- `curl` and `zip` installed.
- Set a few shell variables:

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=<ACCOUNT_ID>
export TABLE=notes-lab
export FN=notes-lab-fn
export ROLE=notes-lab-role
```

## Expected Cost

Effectively $0 within the Free Tier (1M Lambda requests, 400,000 GB-seconds, 25 GB DynamoDB storage, and on-demand requests priced per million). HTTP API costs about $1.00 per million requests after the free tier. CloudWatch Logs ingestion is a few cents at most. Avoid: leaving a load test running, and setting log retention to "never expire" on a busy API.

## Step by Step

### Step 1: Create the DynamoDB table

Console: DynamoDB > Tables > Create table. Name `notes-lab`, partition key `id` (String), Table settings: Customize, capacity mode **On-demand**.

```bash
aws dynamodb create-table \
  --table-name $TABLE \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region $AWS_REGION
```

### Step 2: Create a least-privilege execution role

Console: IAM > Roles > Create role > AWS service > Lambda. Skip managed policies for now, name it `notes-lab-role`, then add the inline policies below.

```bash
cat > trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role --role-name $ROLE \
  --assume-role-policy-document file://trust.json

# Only the two DynamoDB actions this function needs, on this one table
cat > ddb-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["dynamodb:PutItem", "dynamodb:GetItem"],
    "Resource": "arn:aws:dynamodb:$AWS_REGION:$ACCOUNT_ID:table/$TABLE"
  }]
}
EOF
aws iam put-role-policy --role-name $ROLE \
  --policy-name notes-ddb --policy-document file://ddb-policy.json

# Logging to CloudWatch Logs (AWS managed policy)
aws iam attach-role-policy --role-name $ROLE \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

### Step 3: Write and deploy the Lambda function

Console: Lambda > Create function > Author from scratch, name `notes-lab-fn`, runtime Python 3.12, use the existing role `notes-lab-role`. Paste the code in the editor and Deploy. Add environment variable `TABLE_NAME=notes-lab`.

```bash
cat > lambda_function.py <<'EOF'
import json, os, uuid
import boto3

table = boto3.resource("dynamodb").Table(os.environ["TABLE_NAME"])

def handler(event, context):
    route = event["routeKey"]
    if route == "POST /notes":
        body = json.loads(event.get("body") or "{}")
        item = {"id": str(uuid.uuid4()), "text": body.get("text", "")}
        table.put_item(Item=item)
        return {"statusCode": 201, "body": json.dumps(item)}
    if route == "GET /notes/{id}":
        nid = event["pathParameters"]["id"]
        res = table.get_item(Key={"id": nid})
        if "Item" not in res:
            return {"statusCode": 404, "body": json.dumps({"error": "not found"})}
        return {"statusCode": 200, "body": json.dumps(res["Item"])}
    return {"statusCode": 400, "body": "unsupported route"}
EOF
zip function.zip lambda_function.py

# IAM roles can take ~10 seconds to propagate; retry if create-function fails
aws lambda create-function \
  --function-name $FN \
  --runtime python3.12 \
  --handler lambda_function.handler \
  --role arn:aws:iam::$ACCOUNT_ID:role/$ROLE \
  --zip-file fileb://function.zip \
  --environment "Variables={TABLE_NAME=$TABLE}" \
  --region $AWS_REGION
```

### Step 4: Create the HTTP API with a Lambda proxy integration

Console: API Gateway > Create API > HTTP API > Build. Add integration Lambda `notes-lab-fn`. Add routes `POST /notes` and `GET /notes/{id}`. Keep the `$default` stage with auto-deploy.

```bash
API_ID=$(aws apigatewayv2 create-api --name notes-lab-api \
  --protocol-type HTTP --query ApiId --output text)

INT_ID=$(aws apigatewayv2 create-integration --api-id $API_ID \
  --integration-type AWS_PROXY \
  --integration-uri arn:aws:lambda:$AWS_REGION:$ACCOUNT_ID:function:$FN \
  --payload-format-version 2.0 --query IntegrationId --output text)

aws apigatewayv2 create-route --api-id $API_ID \
  --route-key "POST /notes" --target integrations/$INT_ID
aws apigatewayv2 create-route --api-id $API_ID \
  --route-key "GET /notes/{id}" --target integrations/$INT_ID

aws apigatewayv2 create-stage --api-id $API_ID \
  --stage-name '$default' --auto-deploy

# Allow API Gateway to invoke the function (console does this for you)
aws lambda add-permission --function-name $FN \
  --statement-id apigw-invoke --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:$AWS_REGION:$ACCOUNT_ID:$API_ID/*/*"

export API_URL=https://$API_ID.execute-api.$AWS_REGION.amazonaws.com
```

### Step 5: Test the API

```bash
curl -s -X POST $API_URL/notes -H 'content-type: application/json' \
  -d '{"text":"hello serverless"}'
# copy the "id" from the response
curl -s $API_URL/notes/<NOTE_ID>
```

### Step 6: Turn on access logging

Console: API Gateway > your API > Monitor > Logging > `$default` stage > Edit > enable access logging, choose a new log group `/aws/apigw/notes-lab`, and pick the JSON format template.

```bash
aws logs create-log-group --log-group-name /aws/apigw/notes-lab
aws logs put-retention-policy --log-group-name /aws/apigw/notes-lab --retention-in-days 7

aws apigatewayv2 update-stage --api-id $API_ID --stage-name '$default' \
  --access-log-settings "DestinationArn=arn:aws:logs:$AWS_REGION:$ACCOUNT_ID:log-group:/aws/apigw/notes-lab,Format=\$context.requestId \$context.httpMethod \$context.routeKey \$context.status \$context.integrationErrorMessage"
```

If the update is rejected, the log group needs a resource policy letting `delivery.logs.amazonaws.com` write; the console sets this up automatically.

### Step 7: Throttling experiment

Set very low limits on the stage, then fire a burst of requests.

Console: API Gateway > your API > Stages > `$default` > Edit > Default route throttling: rate 1, burst 1.

```bash
aws apigatewayv2 update-stage --api-id $API_ID --stage-name '$default' \
  --default-route-settings ThrottlingBurstLimit=1,ThrottlingRateLimit=1

for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code}\n" $API_URL/notes/does-not-exist &
done; wait
```

You should see a mix of `404` (reached Lambda) and `429` (throttled by API Gateway, Lambda never invoked).

### Step 8: Least-privilege security experiment

Prove the role cannot do more than it should. Temporarily edit the function code so the GET branch calls `table.scan()`, deploy it, and call the route. The call returns a 500 and the Lambda log shows `AccessDeniedException ... not authorized to perform: dynamodb:Scan`. Revert the code afterwards.

Look at the logs:

```bash
aws logs tail /aws/lambda/$FN --since 10m
```

## Validation

- `POST /notes` returns `201` and an item appears in DynamoDB (`aws dynamodb scan --table-name $TABLE --max-items 5`).
- `GET /notes/<NOTE_ID>` returns the same item.
- API access logs show requests with status codes, including `429` during the burst.
- The `dynamodb:Scan` attempt fails with `AccessDeniedException`.

## Cleanup

```bash
aws apigatewayv2 delete-api --api-id $API_ID
aws lambda delete-function --function-name $FN
aws logs delete-log-group --log-group-name /aws/lambda/$FN
aws logs delete-log-group --log-group-name /aws/apigw/notes-lab
aws iam delete-role-policy --role-name $ROLE --policy-name notes-ddb
aws iam detach-role-policy --role-name $ROLE \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name $ROLE
aws dynamodb delete-table --table-name $TABLE
```

## Exam Takeaway

- HTTP API is cheaper and simpler than REST API; choose REST API only when you need features such as API keys with usage plans, request validation, WAF integration or per-client caching.
- The Lambda execution role (what the function can do) is separate from the resource-based permission (who can invoke the function). Both are needed.
- Throttling at API Gateway protects the backend and returns `429` without invoking Lambda. DynamoDB on-demand plus Lambda gives a scale-to-zero, pay-per-request design, the usual answer for unpredictable traffic with minimal operations.
- Scope IAM to specific actions and a specific table ARN rather than `dynamodb:*` on `*`.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
