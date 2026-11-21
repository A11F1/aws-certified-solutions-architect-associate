# Guided Lab: Managed AI Services with Comprehend, Textract and Rekognition

## Objective

Call three pre-trained AWS AI services from the CLI on tiny sample inputs: sentiment and PII detection with Comprehend, text extraction with Textract, and label and moderation detection with Rekognition. Then test what happens without permissions.

## Scenario

A support team wants to classify customer reviews, digitize a scanned receipt and tag uploaded photos, without hiring ML engineers or training models. You use the managed APIs that need no model training.

## Services Used

- Amazon Comprehend
- Amazon Textract
- Amazon Rekognition
- Amazon S3
- AWS IAM
- Optional: Amazon Bedrock

## Prerequisites

- AWS CLI v2 configured for `us-east-1` (all three services are available there).
- A small JPG or PNG photo (a few hundred KB) and a small image or PDF of a receipt or any printed text. Use sample data only, never real personal documents.
- A bucket for the inputs.

```bash
export AWS_REGION=us-east-1
export BUCKET=<YOUR_BUCKET_NAME>
aws s3 mb s3://$BUCKET --region $AWS_REGION
aws s3 cp photo.jpg s3://$BUCKET/inputs/photo.jpg
aws s3 cp receipt.png s3://$BUCKET/inputs/receipt.png
```

## Expected Cost

Pennies. Approximate on-demand prices (verify on the pricing pages): Comprehend sync APIs cost about $0.0001 per 100-character unit (3-unit minimum per request); Textract DetectDocumentText is about $1.50 per 1,000 pages; Rekognition image APIs are about $1.00 per 1,000 images. All three have a Free Tier for the first months of use. Avoid: batch/async jobs, Rekognition Video, Comprehend custom classifiers or endpoints (endpoints bill hourly), and the optional Bedrock step on large prompts.

## Step by Step

### Step 1: Sentiment analysis with Comprehend

Console: Amazon Comprehend > Launch Amazon Comprehend > Real-time analysis > Built-in; paste text and choose Analyze.

```bash
aws comprehend detect-sentiment --language-code en --region $AWS_REGION \
  --text "The delivery was late and the box was damaged, but support fixed it quickly."

aws comprehend detect-dominant-language --region $AWS_REGION \
  --text "Olá, o produto chegou com atraso."

aws comprehend detect-entities --language-code en --region $AWS_REGION \
  --text "Maria Silva bought a laptop in Seattle on March 3rd."
```

Look at the `Sentiment` label and the `SentimentScore` values (Positive, Negative, Neutral, Mixed). The first sentence will likely be `MIXED` or `NEGATIVE`.

### Step 2: Detect PII with Comprehend

```bash
aws comprehend detect-pii-entities --language-code en --region $AWS_REGION \
  --text "Contact John at john@example.com or 555-0100. Card 4111 1111 1111 1111."
```

The response lists entity types and offsets (EMAIL, PHONE, CREDIT_DEBIT_NUMBER, NAME). These offsets can be used to redact text before storing it.

### Step 3: Extract text with Textract

Console: Amazon Textract > Try Textract > Upload document > Raw text.

```bash
aws textract detect-document-text --region $AWS_REGION \
  --document '{"S3Object":{"Bucket":"'$BUCKET'","Name":"inputs/receipt.png"}}' \
  --query 'Blocks[?BlockType==`LINE`].Text' --output text
```

For forms and tables use `analyze-document` with `--feature-types '["FORMS","TABLES"]'`, which costs more per page. For receipts specifically there is `analyze-expense`.

```bash
aws textract analyze-expense --region $AWS_REGION \
  --document '{"S3Object":{"Bucket":"'$BUCKET'","Name":"inputs/receipt.png"}}' \
  --query 'ExpenseDocuments[0].SummaryFields[].[Type.Text,ValueDetection.Text]' --output text
```

Textract's synchronous APIs accept single-page images and PDFs; multi-page PDFs need the asynchronous `start-document-text-detection` flow.

### Step 4: Label and moderation detection with Rekognition

Console: Amazon Rekognition > Label detection > upload the photo.

```bash
aws rekognition detect-labels --region $AWS_REGION \
  --image '{"S3Object":{"Bucket":"'$BUCKET'","Name":"inputs/photo.jpg"}}' \
  --max-labels 10 --min-confidence 80 \
  --query 'Labels[].[Name,Confidence]' --output text

aws rekognition detect-moderation-labels --region $AWS_REGION \
  --image '{"S3Object":{"Bucket":"'$BUCKET'","Name":"inputs/photo.jpg"}}'
```

An empty `ModerationLabels` list means nothing unsafe was detected. Raise or lower `--min-confidence` and notice how the result set changes; this is the threshold decision real applications make.

### Step 5: Permission failure experiment

Create a role or user policy that allows only Comprehend and test that the others are denied. A quick way is a policy simulation.

```bash
cat > ai-policy.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Action":["comprehend:DetectSentiment"],"Resource":"*"}]}
EOF
aws iam create-policy --policy-name ai-lab-comprehend-only \
  --policy-document file://ai-policy.json

aws iam simulate-custom-policy --policy-input-list file://ai-policy.json \
  --action-names comprehend:DetectSentiment rekognition:DetectLabels textract:DetectDocumentText \
  --query 'EvaluationResults[].[EvalActionName,EvalDecision]' --output text
```

Expected: `allowed` for Comprehend, `implicitDeny` for the other two. For Textract and Rekognition reading from S3 you also need `s3:GetObject` on the input object, since the service reads it with your credentials.

### Step 6 (Optional): Amazon Bedrock

Bedrock needs model access enabled per model in your account (Bedrock console > Model access, or in newer console versions the model is enabled on first use, and some models require accepting an end-user agreement or submitting use-case details). Billing is per input and output token, so keep prompts tiny. List what your account can use, then call a model through the Converse API.

```bash
aws bedrock list-foundation-models --region $AWS_REGION \
  --query 'modelSummaries[].modelId' --output text

aws bedrock-runtime converse --region $AWS_REGION \
  --model-id <MODEL_ID_YOU_HAVE_ACCESS_TO> \
  --messages '[{"role":"user","content":[{"text":"Summarize in one sentence: the delivery was late but support fixed it."}]}]' \
  --inference-config '{"maxTokens":100}'
```

Some newer models require an inference profile ID (prefixed `us.`) instead of the base model ID. If you get `AccessDeniedException`, model access is not enabled; that is itself a useful lesson.

## Validation

- Comprehend returns a sentiment label with scores, and PII offsets for the email, phone and card number.
- Textract returns the lines of text you can see in the image.
- Rekognition returns labels above your confidence threshold.
- The policy simulation shows `implicitDeny` for services not in the policy.

## Cleanup

```bash
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ai-lab-comprehend-only
aws s3 rm s3://$BUCKET --recursive
aws s3 rb s3://$BUCKET
```

Nothing else is left running; these APIs are stateless. Revoke Bedrock model access if you enabled it and no longer need it.

## Exam Takeaway

- When the requirement is "add AI with no ML expertise or training", pick the managed AI service for the data type: text (Comprehend), documents (Textract), images/video (Rekognition), speech-to-text (Transcribe), text-to-speech (Polly), translation (Translate), chatbots (Lex).
- Use SageMaker only when you must build, train or host a custom model; use Bedrock for foundation-model/generative use cases without managing infrastructure.
- Comprehend can detect and redact PII, which is a common compliance answer; Textract beats plain OCR when you need forms, tables or expenses.
- Calls to these services are billed per request, so sync APIs suit low-volume, interactive use and async/batch suits large backlogs.

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> This module also connects with cloud data projects; to practice that side, see the complementary track: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
