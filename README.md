### AWS Portfolio Project #5 - Andrew Kremmidas - AWS Solutions Architect Associate 2025

# Serverless Image Processing Pipeline (S3 + Lambda + Rekognition + API Gateway)

You upload an image through a simple web page or API endpoint.

S3 triggers a Lambda function.

Lambda uses Amazon Rekognition to analyze the image (e.g., label detection, moderation, facial analysis).

Lambda writes the results to DynamoDB or S3 as JSON.

A front-end page fetches and displays the labels and confidence scores.


# Demonstrates S3 triggers, Lambda processing, and AI/ML integration with Rekognition.

Uses API Gateway for secure access to analysis results.

Completely serverless — zero EC2, scales automatically.

Recruiters see both application logic and AWS AI service integration.

Core AWS Services

S3 (image upload bucket)

Lambda (processing function)

Amazon Rekognition (image analysis)

API Gateway (REST API for results)

DynamoDB or S3 (store metadata/results)

IAM (least privilege roles)

Workflow

User uploads image → S3 bucket.

S3 event triggers Lambda.

Lambda calls Rekognition → gets labels/faces.

Lambda saves analysis to DynamoDB (or JSON file in another S3 bucket).

API Gateway endpoint returns analysis.


[User Upload]
|
v +-----------------------+
S3 Upload Bucket --+--> Lambda (ingest) ---+--> Rekognition: detect_labels()
| |
| +--> DynamoDB (results)
|
+--> API Gateway --(GET /result?key=...)--> Lambda (get_result) --> DynamoDB


## Services Used
- **Amazon S3** — image uploads
- **AWS Lambda (Python 3.12)** — processing & API
- **Amazon Rekognition** — image label detection (can add moderation/face APIs)
- **Amazon DynamoDB** — result storage
- **Amazon API Gateway (HTTP API)** — public, read-only endpoint
- **Amazon CloudWatch Logs** — logging

## Cost & Safety (Free-tier friendly)
- Keep uploads small (e.g., <5MB test images).
- DynamoDB on-demand with tiny usage.
- Rekognition has per-request charges; test sparingly.
- When Finished, **run the Cleanup** section.

---

### Prereqs & Variables Utilized

- Region: **us-east-1** 
- AWS CLI configured
- Python 3.12 

```bash
# ---- set your variables ----
export REGION=us-east-1
export UPLOAD_BUCKET=andrew-img-uploads-$RANDOM
export TABLE_NAME=img_labels
export PROJECT_TAG=project5-image-pipeline

1) Create S3 upload bucket
aws s3 mb s3://$UPLOAD_BUCKET --region $REGION
# (Optional) add a lifecycle rule or versioning if you want:
aws s3api put-bucket-versioning --bucket $UPLOAD_BUCKET --versioning-configuration Status=Enabled

2) Create DynamoDB table

Primary key = objectKey (String).

aws dynamodb create-table \
  --table-name $TABLE_NAME \
  --attribute-definitions AttributeName=objectKey,AttributeType=S \
  --key-schema AttributeName=objectKey,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region $REGION

aws dynamodb wait table-exists --table-name $TABLE_NAME --region $REGION

3) Create IAM role for Lambda (ingest + get_result)
3.1 Trust policy (assume role)
cat > lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name ImagePipelineLambdaRole \
  --assume-role-policy-document file://lambda-trust-policy.json

3.2 Permissions policy
cat > lambda-inline-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "Logs",
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],
      "Resource": "arn:aws:logs:*:*:*"
    },
    { "Sid": "S3ReadUploadBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject","s3:GetObjectAttributes","s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::*",
        "arn:aws:s3:::*/*"
      ]
    },
    { "Sid": "RekognitionDetect",
      "Effect": "Allow",
      "Action": ["rekognition:DetectLabels"],
      "Resource": "*"
    },
    { "Sid": "DDBWriteRead",
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem","dynamodb:GetItem","dynamodb:DescribeTable"],
      "Resource": "arn:aws:dynamodb:*:*:table/*"
    }
  ]
}
EOF


For production, scope S3 and DynamoDB ARNs tightly to your bucket/table.

aws iam put-role-policy \
  --role-name ImagePipelineLambdaRole \
  --policy-name ImagePipelineInlinePolicy \
  --policy-document file://lambda-inline-policy.json

4) Lambda (ingest) — triggered by S3 object created
4.1 Create function code
mkdir -p lambda/ingest && cd lambda/ingest
cat > app.py << 'EOF'
import os, json, boto3
from urllib.parse import unquote_plus

rek = boto3.client('rekognition')
ddb = boto3.resource('dynamodb')
TABLE_NAME = os.environ.get('TABLE_NAME', 'img_labels')
table = ddb.Table(TABLE_NAME)

def handler(event, context):
    # S3 PUT event
    for record in event.get('Records', []):
        s3 = record.get('s3', {})
        bucket = s3.get('bucket', {}).get('name')
        key = unquote_plus(s3.get('object', {}).get('key'))
        if not bucket or not key:
            continue

        # Rekognition label detection
        resp = rek.detect_labels(
            Image={'S3Object': {'Bucket': bucket, 'Name': key}},
            MaxLabels=10,
            MinConfidence=70.0
        )
        labels = [
            {"Name": l["Name"], "Confidence": float(l["Confidence"])}
            for l in resp.get('Labels', [])
        ]

        # Save to DynamoDB
        item = {
            "objectKey": key,
            "bucket": bucket,
            "labels": labels
        }
        table.put_item(Item=item)
    return {"statusCode": 200, "body": json.dumps({"ok": True})}
EOF
cd ../../

4.2 Zip & create function
cd lambda/ingest
zip -r ../ingest.zip .
cd ..
aws lambda create-function \
  --function-name image_ingest \
  --runtime python3.12 \
  --role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/ImagePipelineLambdaRole \
  --handler app.handler \
  --zip-file fileb://ingest.zip \
  --environment Variables="{TABLE_NAME=$TABLE_NAME}" \
  --tags Project=$PROJECT_TAG \
  --region $REGION
cd ..

5) Wire S3 event to Lambda (ObjectCreated)
# Allow S3 to invoke Lambda
aws lambda add-permission \
  --function-name image_ingest \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --statement-id s3invoke \
  --source-arn arn:aws:s3:::$UPLOAD_BUCKET \
  --region $REGION

# Create event notification on the bucket
cat > s3-notification.json << EOF
{
  "LambdaFunctionConfigurations": [{
    "LambdaFunctionArn": "$(aws lambda get-function --function-name image_ingest --query 'Configuration.FunctionArn' --output text --region $REGION)",
    "Events": ["s3:ObjectCreated:*"]
  }]
}
EOF

aws s3api put-bucket-notification-configuration \
  --bucket $UPLOAD_BUCKET \
  --notification-configuration file://s3-notification.json \
  --region $REGION

6) Lambda (get_result) — API to fetch labels by object key
6.1 Code
mkdir -p lambda/get_result && cd lambda/get_result
cat > app.py << 'EOF'
import os, json, boto3
ddb = boto3.resource('dynamodb')
TABLE_NAME = os.environ.get('TABLE_NAME', 'img_labels')
table = ddb.Table(TABLE_NAME)

def handler(event, context):
    # HTTP API: GET /result?key=<objectKey>
    params = event.get('queryStringParameters') or {}
    key = params.get('key')
    if not key:
        return {"statusCode": 400, "body": "Missing query parameter: key"}
    resp = table.get_item(Key={"objectKey": key})
    item = resp.get('Item')
    if not item:
        return {"statusCode": 404, "body": "Not found"}
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(item)
    }
EOF
cd ../../

6.2 Zip & create function
cd lambda/get_result
zip -r ../get_result.zip .
cd ..
aws lambda create-function \
  --function-name image_get_result \
  --runtime python3.12 \
  --role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/ImagePipelineLambdaRole \
  --handler app.handler \
  --zip-file fileb://get_result.zip \
  --environment Variables="{TABLE_NAME=$TABLE_NAME}" \
  --tags Project=$PROJECT_TAG \
  --region $REGION
cd ..

7) API Gateway (HTTP API)
# Create HTTP API
API_ID=$(aws apigatewayv2 create-api \
  --name image-results-api \
  --protocol-type HTTP \
  --region $REGION \
  --query ApiId --output text)

# Create Lambda integration
LAMBDA_ARN=$(aws lambda get-function --function-name image_get_result \
  --region $REGION --query 'Configuration.FunctionArn' --output text)

INT_ID=$(aws apigatewayv2 create-integration \
  --api-id $API_ID \
  --integration-type AWS_PROXY \
  --integration-uri $LAMBDA_ARN \
  --payload-format-version 2.0 \
  --region $REGION \
  --query IntegrationId --output text)

# Route: GET /result
aws apigatewayv2 create-route \
  --api-id $API_ID \
  --route-key "GET /result" \
  --target integrations/$INT_ID \
  --region $REGION

# Allow API Gateway to invoke Lambda
aws lambda add-permission \
  --function-name image_get_result \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --statement-id apigwinvoke \
  --source-arn "arn:aws:execute-api:$REGION:$(aws sts get-caller-identity --query Account --output text):$API_ID/*/GET/result" \
  --region $REGION

# Deploy (default stage)
aws apigatewayv2 create-stage \
  --api-id $API_ID \
  --stage-name prod \
  --auto-deploy \
  --region $REGION

API_URL=$(aws apigatewayv2 get-api --api-id $API_ID --region $REGION --query 'ApiEndpoint' --output text)
echo "API URL: $API_URL/prod/result?key=<your-object-key>"

8) Test the pipeline
8.1 Upload a test image
curl -o sample.jpg https://picsum.photos/600
aws s3 cp sample.jpg s3://$UPLOAD_BUCKET/


Wait ~5–15 seconds for the event + Rekognition + DynamoDB write.

8.2 Query results
KEY=sample.jpg
curl "$API_URL/prod/result?key=$KEY"


You should get JSON like:

{
  "objectKey": "sample.jpg",
  "bucket": "andrew-img-uploads-12345",
  "labels": [
    {"Name": "Person", "Confidence": 98.12},
    {"Name": "Portrait", "Confidence": 94.33}
  ]
}

9) (Optional) Minimal HTML viewer

Create viewer.html and host anywhere (or open locally):

<!doctype html>
<html>
<head><meta charset="utf-8"><title>Image Labels Viewer</title></head>
<body>
  <h1>Image Labels</h1>
  <input id="key" placeholder="Enter object key (e.g., sample.jpg)" />
  <button onclick="run()">Fetch</button>
  <pre id="out"></pre>
<script>
async function run() {
  const key = document.getElementById('key').value.trim();
  if (!key) return;
  const url = 'REPLACE_WITH_API_URL/prod/result?key=' + encodeURIComponent(key);
  const res = await fetch(url);
  document.getElementById('out').textContent = await res.text();
}
</script>
</body>
</html>


Replace REPLACE_WITH_API_URL with the API_URL printed earlier.

10) Troubleshooting

No result found (404): Confirm the exact object key; check CloudWatch logs for image_ingest.

Lambda timeout/errors: Increase Lambda timeout (e.g., 30s) and check Logs.

Access denied: Re-attach the role or tighten/expand ARNs in the policy to include your bucket/table.

API 500: Check image_get_result logs; ensure TABLE_NAME env matches your DynamoDB table.

11) Cleanup
# API
aws apigatewayv2 delete-stage --api-id $API_ID --stage-name prod --region $REGION
aws apigatewayv2 delete-api --api-id $API_ID --region $REGION

# Lambdas
aws lambda delete-function --function-name image_get_result --region $REGION
aws lambda delete-function --function-name image_ingest --region $REGION

# IAM
aws iam delete-role-policy --role-name ImagePipelineLambdaRole --policy-name ImagePipelineInlinePolicy
aws iam delete-role --role-name ImagePipelineLambdaRole

# DynamoDB
aws dynamodb delete-table --table-name $TABLE_NAME --region $REGION

# S3 (empty then delete)
aws s3 rm s3://$UPLOAD_BUCKET --recursive
aws s3 rb s3://$UPLOAD_BUCKET

# local files
rm -rf lambda s3-notification.json lambda-*.json lambda-*.zip lambda-trust-policy.json lambda-in

