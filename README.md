# AWS Bedrock AgentCore: Autonomous Customer Care Support Agent

[![AWS](https://img.shields.io/badge/AWS-Cloud-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Amazon Bedrock](https://img.shields.io/badge/Amazon_Bedrock-AgentCore_Harness-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/bedrock/)
[![Model](https://img.shields.io/badge/Model-Amazon_Nova_Pro-4285F4?style=for-the-badge)](https://aws.amazon.com/bedrock/nova/)
[![Python](https://img.shields.io/badge/Python-3.9+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![DynamoDB](https://img.shields.io/badge/Amazon-DynamoDB-4053D6?style=for-the-badge&logo=amazondynamodb&logoColor=white)](https://aws.amazon.com/dynamodb/)
[![Evaluation Score](https://img.shields.io/badge/Bedrock_Evaluation-1.00_Correctness-success?style=for-the-badge&logo=checkmarx&logoColor=white)](#automated-evaluation--benchmarking)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

An enterprise-grade, autonomous customer support agent built on the **Amazon Bedrock AgentCore Managed Harness**. The system features **zero-shot intent classification**, **stateful multi-turn parameter collection**, **MCP tool execution** via AWS Lambda and Amazon DynamoDB, **in-prompt domain grounding**, and an automated **LLM-as-a-judge evaluation suite** scoring **1.00 Correctness** across all production paths.

---

## Table of Contents
- [Executive Overview](#executive-overview)
- [System Architecture](#system-architecture)
- [Key Engineering Highlights](#key-engineering-highlights)
- [Visual Proof & Production Verification](#visual-proof--production-verification)
  - [1. Stateful Multi-Turn Bug Filing & Database Persistence](#1-stateful-multi-turn-bug-filing--database-persistence)
  - [2. Knowledge Grounding (Covered vs. Uncovered FAQ)](#2-knowledge-grounding-covered-vs-uncovered-faq)
  - [3. Out-of-Scope Guardrails & Escalation](#3-out-of-scope-guardrails--escalation)
  - [4. Automated Testing & Bedrock Evaluation](#4-automated-testing--bedrock-evaluation)
- [Repository Structure](#repository-structure)
- [Setup & Deployment Guide](#setup--deployment-guide)
  - [Prerequisites](#prerequisites)
  - [Step 1: Deploy Tool Infrastructure (CloudFormation)](#step-1-deploy-tool-infrastructure-cloudformation)
  - [Step 2: Initialize AgentCore Gateway & Register Tools](#step-2-initialize-agentcore-gateway--register-tools)
  - [Step 3: Compile System Prompt & Deploy Harness](#step-3-compile-system-prompt--deploy-harness)
  - [Step 4: Interactive Terminal Chat](#step-4-interactive-terminal-chat)
  - [Step 5: Automated Testing & LLM-as-a-Judge Evaluation](#step-5-automated-testing--llm-as-a-judge-evaluation)
  - [Step 6: Infrastructure Teardown](#step-6-infrastructure-teardown)
- [Automated Evaluation & Benchmarking](#automated-evaluation--benchmarking)
- [License](#license)

---

## Executive Overview

Following AWS's transition from legacy visual Bedrock Flows/Agents Classic to the modern **Amazon Bedrock AgentCore Managed Harness**, this project implements an autonomous conversational support agent. Rather than depending on fragile visual graph editors and hardcoded condition branches, all reasoning, intent classification, memory state management, and tool routing are driven by high-reliability prompt engineering running on top of **Amazon Nova Pro (`us.amazon.nova-pro-v1:0`)**.

### Core Capabilities:
* **Zero-Shot Routing:** Categorizes natural language queries on the fly into three discrete workflows: Bug Reporting, Knowledge Questions, or Out-of-Scope handoffs.
* **Stateful Conversational Memory:** Interactively identifies missing bug report parameters (`description`, `stepsToReproduce`, `environment`) across multiple conversational turns before taking action.
* **Deterministic Tool Invocation:** Interacts with the **AgentCore Gateway** using the Model Context Protocol (MCP) to execute an AWS Lambda function (`create_bug_report`) and persist validated tickets into Amazon DynamoDB.
* **Hallucination-Proof Grounding:** Answers store inquiries strictly against an embedded platform FAQ document; unverified or absent topics are redirected to human telephone support without hallucination.
* **LLM-as-a-Judge Testing:** Benchmarked via an automated evaluation pipeline in **Amazon Bedrock Evaluations**, achieving a **1.00 Correctness** score across all edge cases.

---

## System Architecture

The following diagram illustrates the request routing and execution lifecycle:

```
                      +-----------------------------+
                      |       Customer Query        |
                      +--------------+--------------+
                                     |
                                     v
                 +---------------------------------------+
                 |    Terminal Client (chat.py)          |
                 |  - Preserves runtimeSessionId         |
                 +-------------------+-------------------+
                                     |
                                     v
                 +---------------------------------------+
                 | Amazon Bedrock AgentCore Harness      |
                 | (Model: Amazon Nova Pro @ Greedy T=0) |
                 +-------------------+-------------------+
                                     |
                    Interprets [system_prompt.txt]
                    (Zero-Shot Intent Classification)
                                     |
       +-----------------------------+-----------------------------+
       |                             |                             |
       v                             v                             v
[Route 1: Bug Report]      [Route 2: Platform FAQ]       [Route 3: Out-of-Scope]
Stateful Parameter Check:   Strict Document Retrieval:    Polite Handoff:
- Description               - Validates vs {{FAQ}}        - Refers customer to
- Reproduction Steps        - Denies out-of-doc claims      Human Phone Support
- User Environment          - Answers accurately            (1-800-555-0199)
       |                             |
All params collected?                |
       |                             |
       v                             |
+---------------------------+        |
|   AgentCore Gateway (MCP) |        |
|  - AWS_IAM Authentication |        |
+--------------+------------+        |
               |                     |
               v                     |
+---------------------------+        |
|    AWS Lambda Function    |        |
|   (create_bug_report)     |        |
+--------------+------------+        |
               |                     |
               v                     |
+---------------------------+        |
|     Amazon DynamoDB       |        |
| (Bug Report Ticket Store) |        |
+---------------------------+        |
               |                     |
               +----------+----------+
                          |
                          v
         +----------------------------------+
         | Streaming Response to Customer   |
         | (Ticket ID / Grounded FAQ / Call)|
         +----------------------------------+
```

### Architectural Decisions & Rationale:
1. **AgentCore Managed Harness vs. Visual Flows:** Visual flows introduce vendor-locked configuration drift and lack atomic version control. AgentCore abstracts the agent loop (inference orchestration, memory retention, tool call streaming) into code, making agents testable via CI/CD.
2. **Greedy Decoding for Tool Calling:** The system pins `temperature: 0.0` and `inferenceConfig: {topK: 1}` on Nova Pro, guaranteeing deterministic JSON argument formation for tool execution.
3. **In-Prompt FAQ Grounding:** For compact knowledge domains, embedding canonical markdown directly into the system prompt guarantees sub-second response times without the provisioning latency or overhead of vector databases.

---

## Key Engineering Highlights

| Feature | Implementation Mechanism | Production Impact |
| :--- | :--- | :--- |
| **Intent Routing** | Zero-shot tri-branch classification in `system_prompt.txt` | Eliminates brittle NLP classifiers; instantly handles unstructured phrasing |
| **Stateful Memory** | `runtimeSessionId` state tracking in AgentCore harness | Collects missing parameters across dialog turns without client-side state plumbing |
| **Tool Execution** | AgentCore Gateway via Model Context Protocol (MCP) | Decouples LLM execution from backend infrastructure via fine-grained IAM auth |
| **Data Persistence** | AWS Lambda + Amazon DynamoDB | Ensures durable, auditable ticket tracking with unique UUID ticket IDs |
| **Safety & Grounding**| Strict document constraint (`{{FAQ}}`) | 0% hallucination rate on platform policies; redirects unknown inquiries |
| **Quality Assurance** | Amazon Bedrock Evaluations (LLM-as-a-Judge) | Automated regression gate running test suites across all conversational routes |

---

## Visual Proof & Production Verification

All behaviors have been empirically tested and validated in AWS. Below is visual evidence corresponding to each critical path:

### 1. Stateful Multi-Turn Bug Filing & Database Persistence

When a customer reports an issue without reproduction steps or environment details, the agent retains conversational state, prompts for the missing data one question at a time, and only triggers the database write once all fields are complete:

| Multi-Turn Dialogue & Tool Invocation | DynamoDB Ticket Verification |
| :---: | :---: |
| ![Multi-Turn Bug Filing](assets/workspace_chat_ss.jpg) | ![DynamoDB Ticket Record](assets/dynamo_db_ss.jpg) |
| *Agent asking follow-up questions for steps & environment, then calling `bugreports___create_bug_report`* | *DynamoDB record created with generated Ticket ID, description, steps, environment, and timestamp* |

---

### 2. Knowledge Grounding (Covered vs. Uncovered FAQ)

Platform questions are evaluated strictly against the embedded `online_shop_faq.md`. Questions answerable from documentation receive accurate responses; questions omitted from the FAQ are gracefully redirected to the human support hotline:

| Covered Question (Return Policy) | Uncovered Question (Phone Redirect) |
| :---: | :---: |
| ![FAQ Covered Query](assets/chat_covered_ss.png) | ![FAQ Uncovered Query](assets/chat_uncovered_ss.jpg) |
| *Accurate return policy answer extracted from FAQ* | *Prompt recognizes question is absent in FAQ and escalates to human phone support* |

---

### 3. Out-of-Scope Guardrails & Escalation

Requests completely outside customer support boundaries (e.g., recipe inquiries) are intercepted and redirected to the support hotline without hallucinating or breaking conversational persona:

<div align="center">
  <img src="assets/chat_other_ss.jpg" alt="Out-of-Scope Request Handling" width="700">
  <p><em>Out-of-scope query successfully intercepted with polite escalation to 1-800-555-0199</em></p>
</div>

---

### 4. Automated Testing & Bedrock Evaluation

The agent was benchmarked using Amazon Bedrock Evaluations with Amazon Nova Pro acting as the judge. The test suite (`flow-tests.json`) evaluated all three behavioral pathways:

| Automated Test Suite (`flow-tests.json`) | Bedrock Evaluation Results (1.00 Correctness) |
| :---: | :---: |
| ![Test Suite Definition](assets/harness_tests_ss.jpg) | ![Evaluation Score 1.00](assets/evaluation_ss_1.png) |
| *Automated test harness validating all 3 branches* | *Bedrock Evaluations score: 1.00 (100% pass across all criteria)* |

| Detailed Evaluation Metric Breakdown | System Prompt Configuration |
| :---: | :---: |
| ![Evaluation Summary](assets/evaluation_ss_2.png) | ![System Prompt and FAQ Injection](assets/system_prompt_ss.jpg) |
| *Detailed metrics confirming zero hallucination and exact intent classification* | *System prompt template with dynamic {{FAQ}} injection* |

---

## Repository Structure

```
AWS-Udacity-Customer-Care-Agent-Project/
├── assets/                          # Architecture diagrams & visual proof screenshots
│   ├── chat_covered_ss.png          # FAQ covered question proof
│   ├── chat_other_ss.jpg            # Out-of-scope redirect proof
│   ├── chat_uncovered_ss.jpg        # Uncovered FAQ redirect proof
│   ├── dynamo_db_ss.jpg             # DynamoDB table ticket record proof
│   ├── evaluation_ss_1.png          # Bedrock Evaluation 1.00 score proof
│   ├── evaluation_ss_2.png          # Bedrock Evaluation metrics summary
│   ├── faq_ss.jpg                   # FAQ document configuration screenshot
│   ├── harness_tests_ss.jpg         # Automated test cases run screenshot
│   ├── system_prompt_ss.jpg         # System prompt screenshot
│   └── workspace_chat_ss.jpg        # Multi-turn chat & tool execution proof
├── cloudformation-tool.yaml         # CloudFormation: DynamoDB table, Lambda function, IAM roles
├── cloudformation-testing.yaml      # CloudFormation: Evaluation S3 bucket & Bedrock evaluation role
├── create_bug_report.py             # Lambda handler: validates and writes bug reports to DynamoDB
├── setup_gateway.py                 # Script: Provisions AgentCore Gateway and registers MCP tools
├── create_harness.py                # Script: Injects FAQ, deploys/updates AgentCore Harness
├── chat.py                          # Terminal chat client supporting multi-turn streaming dialog
├── generate-eval-dataset.py         # Script: Runs test cases and generates Bedrock Evaluation dataset
├── cleanup_agentcore.py             # Script: Tears down AgentCore Gateway and Harness resources
├── system_prompt.txt                # Core Deliverable: Zero-shot classification & stateful router prompt
├── online_shop_faq.md               # Canonical FAQ knowledge base for in-prompt grounding
├── flow-tests.json                  # Test suite covering all 3 behavioral routes
├── harness-tests.json               # Test suite configuration for automated runner
├── harness-tests-template.json      # Test suite template definition
├── output_eval_dataset.jsonl        # Evaluation outputs evaluated by Bedrock LLM-as-a-judge
├── agentcore_config.example.json    # Configuration template for AgentCore ARNs and IDs
├── requirements.txt                 # Python dependencies (boto3 >= 1.43, botocore)
├── .gitignore                       # Clean Git ignore rules (AWS credentials, Python bytecode)
├── LICENSE                          # Open-source MIT License
└── README.md                        # Portfolio documentation
```

---

## Setup & Deployment Guide

Follow these steps to replicate and deploy the entire solution in your own AWS account.

### Prerequisites
* **AWS Account:** With access enabled for Amazon Bedrock and Amazon Bedrock AgentCore in region **`us-east-1`**.
* **Model Access:** Amazon Nova Pro (`us.amazon.nova-pro-v1:0`) enabled in the Amazon Bedrock console.
* **AWS CLI:** Installed and configured with appropriate administrator credentials (`aws configure`).
* **Python Environment:** Python 3.9+ installed.

Install the required Python dependencies:
```bash
pip install -r requirements.txt
```

---

### Step 1: Deploy Tool Infrastructure (CloudFormation)

Deploy the backend infrastructure stack, which creates the DynamoDB ticket storage table, the `create_bug_report` AWS Lambda function, the Gateway invocation role, and the Harness execution role:

```bash
aws cloudformation deploy \
  --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

*Note: `--capabilities CAPABILITY_NAMED_IAM` is mandatory because the template provisions named IAM roles with scoped trust relationships.*

---

### Step 2: Initialize AgentCore Gateway & Register Tools

Run `setup_gateway.py`. This script reads the CloudFormation stack outputs, creates an **AgentCore Gateway** (MCP protocol with AWS IAM authentication), registers the Lambda function as the `create_bug_report` tool, and writes configuration metadata to `agentcore_config.json`:

```bash
python setup_gateway.py
```

*Output summary:*
```
Created gateway: bug-report-tool-stack-gateway
Registered gateway target: bugreports (Tool: create_bug_report)
Config written to agentcore_config.json
```

---

### Step 3: Compile System Prompt & Deploy Harness

`create_harness.py` reads `system_prompt.txt`, dynamically replaces the `{{FAQ}}` placeholder with the contents of `online_shop_faq.md`, and provisions (or updates) the managed harness running on **Amazon Nova Pro**:

```bash
python create_harness.py
```

*Features:*
* Automatically retries on IAM propagation delays.
* Idempotent: re-running updates the existing harness without provisioning a new one.
* Pinned model: `us.amazon.nova-pro-v1:0` with greedy decoding (`temperature: 0.0`, `topK: 1`).

---

### Step 4: Interactive Terminal Chat

Launch the interactive chat client to test multi-turn conversations:

```bash
python chat.py
```

**Try out the different routes:**
1. **Bug Report (Multi-turn parameter collection):**
   ```text
   You: The checkout button doesn't work.
   Bot: I'm sorry to hear that. Could you please provide the steps to reproduce the issue?
   You: 1. Add item to cart 2. Click Checkout
   Bot: Thank you. What environment are you using (browser, operating system, and device)?
   You: Chrome on macOS Sonoma
   Bot: [tool call] bugreports___create_bug_report ...
        Your bug report has been submitted! Ticket ID: 4dbe1615-9d2e-495d-9ebb-472def05f695.
   ```
2. **Grounded FAQ:**
   ```text
   You: What is your return policy?
   Bot: You can return most items within 30 days of delivery as long as they are unused...
   ```
3. **Out-of-Scope Redirect:**
   ```text
   You: Can you write a poem about artificial intelligence?
   Bot: I'm sorry, but I can't help with that here. Please reach out to our human support line at 1-800-555-0199 (Mon-Fri).
   ```

Verify created tickets in DynamoDB:
```bash
aws dynamodb scan \
  --table-name bug-report-tool-stack-bug-reports \
  --region us-east-1
```

---

### Step 5: Automated Testing & LLM-as-a-Judge Evaluation

1. **Deploy Evaluation Infrastructure:**
   Deploy the evaluation stack to provision the dedicated S3 dataset bucket and Bedrock evaluation IAM execution role:
   ```bash
   aws cloudformation deploy \
     --template-file cloudformation-testing.yaml \
     --stack-name bug-report-testing-stack \
     --capabilities CAPABILITY_NAMED_IAM \
     --region us-east-1
   ```

2. **Generate Evaluation Dataset:**
   Execute the test suite (`flow-tests.json` / `harness-tests.json`) through the automated generator:
   ```bash
   python generate-eval-dataset.py --tests-json flow-tests.json
   ```
   This executes each test case against a fresh agent session and writes formatted responses to `output_eval_dataset.jsonl`.

3. **Run Bedrock Evaluation Job:**
   * Upload `output_eval_dataset.jsonl` to the evaluation S3 bucket.
   * Open the **Amazon Bedrock Console** -> **Assessments & evaluations** -> **Create evaluation job**.
   * Select **LLM-as-a-judge** with **Amazon Nova Pro** as the evaluator.
   * Target Metric: **Correctness**.
   * Run the evaluation job to produce the **1.00 Correctness** score.

---

### Step 6: Infrastructure Teardown

To cleanly remove all AWS resources and avoid unexpected charges, execute the cleanup script and delete CloudFormation stacks:

```bash
# 1. Delete AgentCore harness, gateway target, and gateway
python cleanup_agentcore.py

# 2. Delete the tool infrastructure stack
aws cloudformation delete-stack --stack-name bug-report-tool-stack --region us-east-1

# 3. Empty and delete the testing stack S3 bucket
aws s3 rm s3://udacity-agentic-engineer-c1-eval-<YOUR_ACCOUNT_ID> --recursive
aws cloudformation delete-stack --stack-name bug-report-testing-stack --region us-east-1
```

---

## Automated Evaluation & Benchmarking

The automated evaluation framework uses the **LLM-as-a-Judge** methodology to eliminate subjective manual testing. The test suite evaluates prompt adherence against standard test criteria:

| Test ID | Input Prompt | Expected Evaluation Behavior | Result |
| :--- | :--- | :--- | :---: |
| `t1_faq_return` | *"How many days do I have to return an item?"* | Bot answers accurately using strictly the 30-day window defined in `online_shop_faq.md`. | **PASS (1.00)** |
| `t2_redirect_out_of_scope` | *"Can you give me a good recipe for chocolate chip cookies?"* | Bot identifies query as out-of-scope and redirects customer to `1-800-555-0199`. | **PASS (1.00)** |
| `t3_bug_missing_info` | *"The checkout page is broken."* | Bot classifies intent as bug report but does **not** call the tool; requests missing parameters. | **PASS (1.00)** |
| `t4_bug_complete_info` | *"I want to report a bug. The search bar crashes. Steps: 1. Go to homepage... Env: Firefox on Windows 11."* | Bot extracts all 3 fields, calls `create_bug_report`, and returns the created Ticket ID. | **PASS (1.00)** |

**Aggregate Bedrock Evaluation Correctness Score: 1.00 / 1.00 (100%)**

---

## License

This project is licensed under the [MIT License](LICENSE) - see the LICENSE file for details.