# Review Questions

## Question 1

A scenario requires the right decision on ML and AI. Which approach fits best?

A) Apply a generic solution without considering SageMaker.
B) Use SageMaker with configuration driven by the requirement and lean operations.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of ML and AI with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
A generic answer does not take SageMaker into account, so it misses the trade-off the question is testing.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 2

Which choice reduces operational risk in this domain?

A) Handle everything manually during an incident.
B) Combine inference with observability and automation.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of ML and AI with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Manual handling is slow, error-prone, and does not scale. Combining automation with observability is what actually reduces operational risk.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 3

What should you do to keep cost under control without losing technical quality?

A) Always pick the cheapest option without validating the requirement.
B) Size based on actual usage and take advantage of drift when it makes sense.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of ML and AI with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Cost is only one constraint. The cheapest option can break availability, performance, or security requirements, so it must be validated against the scenario first.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 4

Which alternative tends to be incorrect on the exam for this topic?
A) The one that meets the dominant requirement with the least complexity.
B) The one that ignores modelo pre-treinado signals and architecture dependencies.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of ML and AI with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
This is the pattern that is usually correct on the exam. The question asks which alternative tends to be incorrect, so this is the opposite of what is being asked.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 5

Which review approach improves accuracy in this module?

A) Review only definitions without scenario practice.
B) Practice questions focused on latencia and cost with explanations of the trade-offs.
C) Add components the question does not call for.
D) Ignore an explicit security, availability, or cost requirement.

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Option B covers the technical goal of ML and AI with the best balance between requirement, risk, and operations.

**Why option A is wrong:**
Definitions alone do not build the judgment needed to choose between two plausible options. Scenario practice with trade-off explanations does.

**Why option C is wrong:**
Extra components add cost and operational complexity without addressing any requirement stated in the question. On the exam, the best answer is the simplest design that meets the requirement.

**Why option D is wrong:**
The question states this requirement explicitly, so an option that ignores it fails the scenario no matter what else it offers.

</details>

## Question 6

A call center stores thousands of recorded customer calls as audio files in Amazon S3. Compliance requires a searchable text transcript of every call, with customer names, phone numbers, and Social Security numbers automatically masked in the transcript. The team has no machine learning expertise and wants a fully managed solution. Which approach meets these requirements?

A) Use Amazon Rekognition to analyze the audio files and a Lambda function to remove sensitive values
B) Use Amazon Transcribe with a custom vocabulary that lists the sensitive terms to hide
C) Run Amazon Transcribe batch jobs with content redaction enabled for personally identifiable information
D) Send the audio files directly to Amazon Comprehend and use its PII detection to produce redacted transcripts

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Amazon Transcribe converts speech to text and offers built-in content redaction for PII, replacing sensitive values in the transcript with a placeholder. It is fully managed, reads audio from S3, and needs no ML knowledge. Option C produces both the transcript and the redaction in one step.

**Why option A is wrong:**
Amazon Rekognition analyzes images and video (objects, faces, text in frames), not speech in audio files. Building redaction logic in Lambda would also be unnecessary custom work.

**Why option B is wrong:**
A custom vocabulary improves recognition accuracy for domain-specific words such as product names. It does not hide or mask any content in the output.

**Why option D is wrong:**
Amazon Comprehend operates on text, not audio, so the calls would have to be transcribed first. Comprehend can detect PII in text, but Transcribe already offers redaction as part of the transcription job.

</details>

## Question 7

An online retailer wants to show each shopper personalized product recommendations on the home page, based on browsing and purchase history. Recommendations must update within seconds as the shopper clicks around. The company has no data scientists and wants to avoid building and tuning models itself. Which service should the solutions architect choose?

A) Amazon Personalize
B) Amazon SageMaker with a custom recommendation algorithm built in a notebook
C) Amazon Comprehend with custom classification
D) Amazon Kendra with an index of the product catalog

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** A

**Explanation:**
Amazon Personalize is a managed service that builds recommendation models from user interaction data and serves real-time recommendations through an API. It can incorporate new events as they arrive, and it does not require ML expertise. Option A matches the requirement to avoid building models in-house.

**Why option B is wrong:**
SageMaker can certainly build a recommender, but the team would have to prepare data, choose and tune algorithms, and operate the endpoint. That is exactly the effort the company wants to avoid.

**Why option C is wrong:**
Comprehend analyzes text for sentiment, entities, and topics. It does not generate item recommendations from behavioral data.

**Why option D is wrong:**
Kendra is an enterprise search service that answers natural-language queries over documents. It does not personalize a home page based on a user's behavior.

</details>

## Question 8

A data science team must train a custom deep learning model on 2 TB of data stored in S3. Training uses GPU instances, runs for about 20 hours, and can safely resume from checkpoints if interrupted. Management wants the lowest possible training cost without the team managing servers or remembering to shut them down. What should the architect recommend?

A) Launch On-Demand GPU EC2 instances and use a script to stop them when training finishes
B) Run the training code in AWS Lambda functions triggered by S3 events
C) Deploy the model to a SageMaker real-time endpoint and train it with incoming requests
D) Use Amazon SageMaker training jobs with managed spot training and checkpointing to S3

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** D

**Explanation:**
SageMaker training jobs provision the instances, run the training, and terminate the instances automatically when the job ends. Managed spot training uses spare EC2 capacity at a steep discount, and checkpoints in S3 let an interrupted job resume. Option D gives the lowest cost with no server management.

**Why option A is wrong:**
On-Demand pricing is the most expensive choice, and relying on a shutdown script reintroduces manual operations and the risk of leaving costly GPU instances running.

**Why option B is wrong:**
Lambda has a 15-minute maximum duration, no GPU support, and limited memory, so it cannot run a 20-hour GPU training job.

**Why option C is wrong:**
A real-time endpoint serves inference for an already trained model and bills continuously while running. It is not a mechanism for training a model.

</details>

## Question 9

An insurance company receives scanned claim forms as PDFs and JPEG images. It needs to extract printed and handwritten text, key-value pairs such as policy number and claim amount, and tables, then store the structured results in DynamoDB. The solution should be serverless and require no model training. Which service performs the extraction?

A) Amazon Rekognition text detection
B) Amazon Textract
C) Amazon Comprehend
D) Amazon Translate

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** B

**Explanation:**
Amazon Textract goes beyond basic OCR: it extracts text, handwriting, form key-value pairs, and tables from scanned documents, with no training required. Its output can be written to DynamoDB by a Lambda function. Option B is the purpose-built document extraction service.

**Why option A is wrong:**
Rekognition text detection reads short text in images such as signs or license plates. It does not understand form structure, key-value pairs, or tables, and it is not designed for multi-page documents.

**Why option C is wrong:**
Comprehend analyzes text that already exists (entities, sentiment, language). It cannot read scanned images or PDFs of forms.

**Why option D is wrong:**
Amazon Translate converts text between languages. It does not extract data from document images.

</details>

## Question 10

A software company wants to add a generative AI writing assistant to its product. Developers want API access to several foundation models, and the security team requires that customer prompts stay private and are not used to train the underlying base models. The company does not want to provision or manage GPU infrastructure. Which solution fits?

A) Deploy an open-source model from SageMaker JumpStart on a real-time endpoint that the team scales and patches
B) Train a large language model from scratch on a cluster of GPU EC2 instances
C) Use Amazon Bedrock to invoke foundation models through its API
D) Build the assistant with Amazon Lex, which hosts the foundation models for text generation

<details>
<summary><strong>Show answer</strong></summary>

✅ **Correct answer:** C

**Explanation:**
Amazon Bedrock is a fully managed, serverless service that exposes foundation models from several providers through a single API. Customer inputs and outputs are not used to train the base models, and data can be protected with encryption and VPC endpoints. Option C removes the need to manage GPU infrastructure.

**Why option A is wrong:**
JumpStart can deploy open-source models, but the endpoint runs on instances the company must size, scale, and patch. That contradicts the requirement to avoid infrastructure management.

**Why option B is wrong:**
Training an LLM from scratch is extremely expensive and time-consuming, requires large amounts of data and ML expertise, and is far beyond what the use case needs.

**Why option D is wrong:**
Amazon Lex is a service for building conversational interfaces such as chatbots with intents and slots. It is not a catalog of foundation models for general text generation.

</details>

---

## ☁️ Follow CloudStudy

We are building a platform to help Brazilians study AWS in a more practical, organized, and accessible way.

Follow CloudStudy to keep up with new materials, updates, and content about AWS certifications:

- Instagram: https://www.instagram.com/cloudstudy.ai/
- LinkedIn: https://www.linkedin.com/company/cloudstudy-ai/

---

> To consolidate architecture with real data projects on AWS, keep the complementary track on your radar: [Data Engineering on AWS](https://www.udemy.com/course/engenharia-de-dados-na-aws-do-zero-aos-projetos-reais/?referralCode=E28670B9116BA68E08A9).
