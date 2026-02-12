# Design Document: AI-Powered Adaptive Learning Platform

## 1. System Overview

The AI-Powered Adaptive Learning Platform is a serverless, AWS-native intelligent learning system designed to provide personalized, adaptive education at scale. The platform continuously diagnoses student understanding, adapts content delivery in real-time, and provides explainable AI reasoning to build trust and improve learning outcomes.

### Core Design Principles

1. **Adaptive Intelligence**: Every interaction is informed by real-time proficiency tracking stored in DynamoDB
2. **Explainability First**: All AI decisions are accompanied by transparent reasoning through the XAI module
3. **Serverless Architecture**: 100% serverless design using Lambda, Step Functions, and managed services for automatic scaling
4. **Cost Optimization**: Aggressive caching, pre-rendered templates, and Spot Instances minimize operational costs
5. **Modular Design**: Four independent modules (Core Tutor, Depth Tools, Visual & Challenge, Planning) orchestrated via Step Functions

### Key Differentiators

- **Real-time Proficiency Tracking**: Unlike static learning platforms, every interaction updates the student's proficiency model
- **Multi-modal Learning**: Seamlessly switches between text, visual animations, code execution, and research tools
- **Context-Aware Research**: Web searches and citations filtered by current mindmap context to prevent information overload
- **Exam-Focused Planning**: AI-generated study schedules based on syllabus extraction and past paper analysis
- **Production-Ready**: Enterprise-grade architecture with monitoring, tracing, and auto-scaling from day one

## 2. AWS-Native Architecture Rationale

### Why 100% AWS Services?

1. **Unified Ecosystem**: Single cloud provider simplifies IAM, networking, and service integration
2. **Managed Services**: DynamoDB, Bedrock, and Textract eliminate infrastructure management overhead
3. **Auto-Scaling**: Lambda and DynamoDB automatically handle load variations without manual intervention
4. **Cost Efficiency**: Pay-per-use pricing with generous free tiers reduces demo phase costs to $26-32/month
5. **Hackathon Alignment**: Demonstrates deep AWS expertise and maximizes "AWS-native usage" scoring criteria


### Service Selection Justification

| Service | Purpose | Alternative Considered | Why AWS Service Chosen |
|---------|---------|----------------------|----------------------|
| AWS Bedrock | AI inference for all agents | OpenAI API, Anthropic Direct | Native AWS integration, prompt caching, batch API for cost savings |
| AWS Textract | OCR for handwritten notes | Google Vision API, Tesseract | Superior handwriting recognition, native AWS integration, no external API calls |
| AWS Step Functions | Workflow orchestration | Apache Airflow, custom Lambda chains | Visual workflow diagrams for demos, built-in retry logic, native AWS integration |
| DynamoDB | Proficiency and user data | RDS PostgreSQL, MongoDB Atlas | Serverless, auto-scaling, single-digit millisecond latency, no server management |
| Lambda | Agent execution | EC2, ECS Fargate | Pay-per-invocation, automatic scaling, zero server management |
| S3 + CloudFront | Video delivery | YouTube, Vimeo | Full control, no third-party dependencies, global CDN with sub-second latency |
| AWS Cognito | Authentication | Auth0, Firebase Auth | Native AWS integration, no external dependencies, built-in user pools |
| CloudWatch + X-Ray | Monitoring and tracing | Datadog, New Relic | Native integration, no additional cost for basic monitoring, unified logging |

## 3. High-Level Architecture

### Layered Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                          │
│  React SPA (AWS Amplify) + CloudFront CDN                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        API LAYER                                │
│  API Gateway (REST + WebSocket) + AWS Cognito                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   ORCHESTRATION LAYER                           │
│  AWS Step Functions (Workflow Engine)                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      AGENT LAYER                                │
│  Lambda Functions (Diagnostic, Explanation, OCR, Citation,      │
│  Mindmap, Coding, Web Search, Visualization, Challenge,         │
│  Exam Planner, XAI)                                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    AI/ML SERVICES LAYER                         │
│  AWS Bedrock (Claude 3.5 Sonnet/Haiku) + AWS Textract         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                 │
│  DynamoDB (Proficiency, Users, Sessions, Mindmaps, Citations)  │
│  S3 (Videos, Images, Documents)                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   MONITORING LAYER                              │
│  CloudWatch (Logs, Metrics, Alarms) + X-Ray (Tracing)         │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow Architecture

1. **Student Request**: Browser → API Gateway → Cognito (auth) → Step Functions
2. **Orchestration**: Step Functions routes to appropriate Lambda agents based on request type
3. **Agent Execution**: Lambda agents call Bedrock/Textract, read/write DynamoDB, store artifacts in S3
4. **Response Aggregation**: Step Functions collects agent outputs, returns unified response
5. **Content Delivery**: Static assets via CloudFront, videos via S3+CloudFront CDN
6. **Monitoring**: All operations logged to CloudWatch, traced via X-Ray

## 4. Component Design

### 4.1 Module 1: Core Tutor

#### 4.1.1 Diagnostic Agent

**Purpose**: Analyze student responses to determine current proficiency level

**Implementation**:
- Lambda function (Python 3.11, 512MB memory, 30s timeout)
- Uses AWS Bedrock Claude 3.5 Sonnet for response analysis
- Prompt engineering: "Analyze this student response and rate understanding from 0-100 based on correctness, depth, and clarity"

**Inputs**:
- Student response (text)
- Question context
- Previous proficiency score (from DynamoDB)

**Outputs**:
- Proficiency score (0-100)
- Confidence level (0-1)
- Reasoning for score

**DynamoDB Schema**:
```json
{
  "PK": "USER#<user_id>",
  "SK": "PROFICIENCY#<topic_id>#<timestamp>",
  "proficiency_score": 75,
  "confidence": 0.85,
  "reasoning": "Student demonstrated solid understanding...",
  "topic": "calculus_derivatives",
  "timestamp": "2024-02-15T10:30:00Z"
}
```

**Error Handling**:
- If Bedrock call fails: Retry 3 times with exponential backoff
- If all retries fail: Return previous proficiency score with confidence 0.5
- Log all errors to CloudWatch


#### 4.1.2 Explanation Agent

**Purpose**: Generate adaptive explanations based on student proficiency level

**Implementation**:
- Lambda function (Python 3.11, 1024MB memory, 60s timeout)
- Uses AWS Bedrock Claude 3.5 Sonnet with prompt caching
- Adaptive prompt templates based on proficiency ranges:
  - Low (0-40): "Explain using simple language, basic examples, avoid jargon"
  - Medium (40-70): "Explain using intermediate terminology, moderate depth"
  - High (70-100): "Explain using advanced terminology, comprehensive depth, include edge cases"

**Inputs**:
- Question/topic
- Student proficiency score
- Conversation history (last 5 messages)

**Outputs**:
- Adaptive explanation (text)
- Suggested follow-up questions
- Related topics for exploration

**Prompt Caching Strategy**:
- Cache system prompt (defines agent role and behavior)
- Cache topic context (definitions, formulas, examples)
- Only variable: student question and proficiency level
- Expected cache hit rate: 60%+

**Performance Optimization**:
- Use Claude 3 Haiku for demo phase (83% cheaper, $0.25/M input tokens)
- Upgrade to Claude 3.5 Sonnet for production ($3/M input tokens)
- Batch API for non-real-time requests (50% cost reduction)

#### 4.1.3 OCR Agent

**Purpose**: Extract text from handwritten notes and images

**Implementation**:
- Lambda function (Python 3.11, 2048MB memory, 30s timeout)
- Uses AWS Textract DetectDocumentText API
- Image preprocessing: resize to max 5MB, convert to PNG if needed

**Inputs**:
- Image file (JPEG, PNG, PDF)
- Image metadata (upload timestamp, student ID)

**Outputs**:
- Extracted text
- Confidence scores per text block
- Bounding box coordinates

**Processing Flow**:
1. Validate image format and size
2. Upload to S3 temporary bucket (30-day lifecycle policy)
3. Call Textract DetectDocumentText
4. Parse response, filter blocks with confidence < 60%
5. Return extracted text with confidence scores
6. If average confidence < 60%, request re-upload

**Cost Optimization**:
- Use Textract only for handwritten/complex images
- For printed text, use lighter OCR library (Tesseract) first
- Estimated cost: $1.50 per 1000 pages

#### 4.1.4 XAI Module

**Purpose**: Provide explainable reasoning for AI decisions

**Implementation**:
- Lambda function (Python 3.11, 512MB memory, 20s timeout)
- Uses AWS Bedrock Claude 3.5 Sonnet with specialized XAI prompt
- Generates natural language explanations for:
  - Why a specific hint was provided
  - Why a learning path was suggested
  - How proficiency score was calculated

**Inputs**:
- AI decision (hint, learning path, proficiency score)
- Student proficiency history
- Current learning context

**Outputs**:
- Natural language explanation
- Supporting evidence (proficiency scores, learning patterns)
- Actionable insights for student

**XAI Prompt Template**:
```
You are an educational AI explainer. A student received the following recommendation: {decision}

Based on their proficiency history: {proficiency_data}
And current learning context: {context}

Explain in simple terms:
1. Why this recommendation was made
2. What data informed this decision
3. How the student can use this information to improve

Keep explanation under 100 words, use encouraging tone.
```

### 4.2 Module 2: Depth Tools

#### 4.2.1 Citation Researcher

**Purpose**: Search and retrieve academic citations from scholarly databases

**Implementation**:
- Lambda function (Python 3.11, 512MB memory, 30s timeout)
- Integration with Google Scholar API (or Semantic Scholar API as fallback)
- Results stored in DynamoDB for caching

**Inputs**:
- Search query
- Number of results (default: 5)
- Filters (year range, citation count threshold)

**Outputs**:
- List of citations with metadata:
  - Title
  - Authors
  - Publication year
  - Citation count
  - Abstract (first 200 chars)
  - DOI link
  - PDF link (if available)

**DynamoDB Schema**:
```json
{
  "PK": "CITATION#<citation_id>",
  "SK": "METADATA",
  "title": "Deep Learning for Education",
  "authors": ["Smith, J.", "Doe, A."],
  "year": 2023,
  "citation_count": 145,
  "abstract": "This paper explores...",
  "doi": "10.1234/example",
  "pdf_url": "https://...",
  "bookmarked_by": ["user1", "user2"],
  "ttl": 1234567890
}
```

**Caching Strategy**:
- Cache search results for 7 days (TTL in DynamoDB)
- Cache hit rate expected: 40%+
- Reduces API calls and improves response time

#### 4.2.2 Mindmap Generator

**Purpose**: Create interactive concept graphs from conversation or topic

**Implementation**:
- Lambda function (Python 3.11, 1024MB memory, 45s timeout)
- Uses AWS Bedrock to extract concepts and relationships
- Generates D3.js-compatible graph structure

**Inputs**:
- Conversation history or topic description
- Existing mindmap (if updating)

**Outputs**:
- Graph structure (nodes and edges)
- Node metadata (concept name, description, links)
- Rendering instructions for D3.js

**Concept Extraction Prompt**:
```
Extract key concepts and their relationships from this text: {text}

Return JSON format:
{
  "nodes": [
    {"id": "concept1", "label": "Calculus", "description": "...", "links": ["wikipedia_url", "citation_id"]}
  ],
  "edges": [
    {"source": "concept1", "target": "concept2", "relationship": "prerequisite"}
  ]
}
```

**Graph Storage**:
```json
{
  "PK": "USER#<user_id>",
  "SK": "MINDMAP#<mindmap_id>",
  "graph": {
    "nodes": [...],
    "edges": [...]
  },
  "created_at": "2024-02-15T10:30:00Z",
  "updated_at": "2024-02-15T11:00:00Z"
}
```

**Frontend Rendering**:
- D3.js force-directed graph
- Interactive: click nodes to expand, drag to rearrange
- Color-coded by topic category
- Links open in modal overlay


#### 4.2.3 Coding Agent

**Purpose**: Execute student code and run tests in secure sandbox

**Implementation**:
- Lambda function (Python 3.11, 3008MB memory, 5 min timeout)
- Secure sandbox using Lambda execution environment
- Supports Python, JavaScript, Java (via Lambda layers)

**Inputs**:
- Source code
- Test cases
- Language/runtime

**Outputs**:
- Test results (pass/fail)
- Execution output (stdout/stderr)
- Error messages and stack traces
- AI-generated fix suggestions (if tests fail)

**Security Measures**:
- No network access from sandbox
- Resource limits: 3GB memory, 5 min timeout
- Restricted file system access
- No access to AWS credentials

**Test Execution Flow**:
1. Validate code syntax
2. Create isolated execution environment
3. Run code with test inputs
4. Capture output and compare with expected results
5. If tests fail, call Bedrock to analyze and suggest fixes
6. Return results with AI suggestions

**Fix Suggestion Prompt**:
```
Student code failed tests:
Code: {code}
Test: {test_case}
Expected: {expected_output}
Actual: {actual_output}
Error: {error_message}

Provide:
1. Explanation of what went wrong
2. Specific fix suggestion
3. Corrected code snippet

Adapt explanation to proficiency level: {proficiency_score}
```

#### 4.2.4 Web Search Agent

**Purpose**: Perform contextual web searches filtered by learning context

**Implementation**:
- Lambda function (Python 3.11, 512MB memory, 30s timeout)
- Uses Google Custom Search API or Bing Search API
- Filters results based on current mindmap context

**Inputs**:
- Search query
- Current mindmap context (optional)
- Number of results (default: 5)

**Outputs**:
- Filtered search results
- AI-synthesized summary of top results
- Source citations

**Context Filtering Algorithm**:
1. Extract concepts from current mindmap
2. Expand search query with related concepts
3. Perform web search
4. Score results based on concept overlap
5. Return top N results with highest scores
6. Use Bedrock to synthesize key information

**Synthesis Prompt**:
```
Synthesize information from these search results: {results}

Focus on concepts: {mindmap_concepts}

Provide:
1. Key takeaways (3-5 bullet points)
2. How this relates to current learning context
3. Recommended next steps

Cite sources for each point.
```

### 4.3 Module 3: Visual & Challenge Layer

#### 4.3.1 Visualization Engine

**Purpose**: Generate 3Blue1Brown-style mathematical and algorithmic animations

**Implementation**:
- Lambda function (Python 3.11, 512MB memory, 60s timeout) for code generation
- EC2 Spot Instances (t3.medium or g4dn.xlarge) for Manim rendering
- S3 for video storage, CloudFront for delivery

**Inputs**:
- Topic/concept to visualize
- Learning goal
- Student proficiency level

**Outputs**:
- Manim Python code
- Rendered video (MP4)
- CloudFront URL for streaming

**Manim Code Generation Prompt**:
```
Generate Manim code to visualize: {concept}

Learning goal: {goal}
Student level: {proficiency_level}

Requirements:
1. Clear, step-by-step animation
2. Use colors to highlight key elements
3. Include text annotations
4. Duration: 30-60 seconds
5. Resolution: 1080p

Return only valid Manim code, no explanations.
```

**Rendering Pipeline**:
1. Generate Manim code using Bedrock
2. Validate code syntax
3. Submit rendering job to EC2 Spot Instance via SQS queue
4. EC2 instance:
   - Pulls job from SQS
   - Executes Manim rendering
   - Uploads video to S3
   - Sends completion notification via SNS
5. Lambda receives SNS notification, updates DynamoDB
6. Return CloudFront URL to student

**Pre-rendered Template Strategy**:
- Identify 50 most common topics (calculus, sorting algorithms, etc.)
- Pre-render animations for these topics
- Store in S3 with CloudFront caching
- 95% cost reduction for common topics
- Custom rendering only for unique requests

**Cost Optimization**:
- EC2 Spot Instances: 70% cheaper than on-demand
- Spot interruption handling: Save partial renders to S3, resume on new instance
- Video compression: H.264 codec, reduce file size by 50%+
- CloudFront caching: 24-hour TTL, reduce S3 GET requests

#### 4.3.2 User Preference Tracking

**Purpose**: Track student preferences for visual vs text learning

**Implementation**:
- DynamoDB table for preference storage
- Lambda function for preference analysis

**DynamoDB Schema**:
```json
{
  "PK": "USER#<user_id>",
  "SK": "PREFERENCES",
  "visual_count": 15,
  "text_count": 10,
  "visual_preference_score": 0.6,
  "preferred_topics_for_visual": ["math", "algorithms"],
  "last_updated": "2024-02-15T10:30:00Z"
}
```

**Preference Analysis**:
- Calculate visual_preference_score = visual_count / (visual_count + text_count)
- If score > 0.6, proactively suggest visualizations
- If score < 0.4, default to text explanations
- Track preferences per topic category


#### 4.3.3 Challenge Generator

**Purpose**: Create personalized daily practice problems with escalating difficulty

**Implementation**:
- Lambda function (Python 3.11, 1024MB memory, 60s timeout)
- Uses AWS Bedrock to generate problems
- Applies spaced repetition algorithm for weak topics

**Inputs**:
- Student proficiency data across topics
- Daily challenge goal (number of problems)
- Weak topics (proficiency < 50)

**Outputs**:
- List of practice problems
- Difficulty distribution (60% current level, 30% harder, 10% significantly harder)
- Spaced repetition schedule for weak topics

**Problem Generation Prompt**:
```
Generate {count} practice problems for topic: {topic}

Difficulty level: {difficulty}
Student proficiency: {proficiency_score}

Requirements:
1. Problems should test understanding, not just memorization
2. Include variety: multiple choice, short answer, coding challenges
3. Provide correct answers and explanations
4. Ensure problems are culturally relevant to Indian students

Return JSON format:
{
  "problems": [
    {
      "question": "...",
      "type": "multiple_choice",
      "options": ["A", "B", "C", "D"],
      "correct_answer": "B",
      "explanation": "..."
    }
  ]
}
```

**Spaced Repetition Algorithm**:
- Based on SM-2 algorithm (SuperMemo)
- Intervals: 1 day, 3 days, 7 days, 14 days, 30 days
- If student fails problem, reset interval to 1 day
- If student succeeds, advance to next interval

**DynamoDB Schema**:
```json
{
  "PK": "USER#<user_id>",
  "SK": "CHALLENGE#<date>",
  "problems": [...],
  "completed": false,
  "score": null,
  "completion_time": null
}
```

### 4.4 Module 4: Planning & Self-Regulation

#### 4.4.1 Exam Planner

**Purpose**: Generate AI-powered study schedules based on syllabus and exam date

**Implementation**:
- Lambda function (Python 3.11, 1024MB memory, 90s timeout)
- Uses AWS Textract for syllabus extraction
- Uses AWS Bedrock for schedule generation

**Inputs**:
- Exam date
- Available daily study time (hours)
- Target performance level (pass, good, excellent)
- Syllabus PDF
- Past exam papers (optional)

**Outputs**:
- Daily study schedule from now to exam date
- Topic allocation based on weights and proficiency
- Review sessions for weak topics
- Rest days and buffer time
- Visual learning path

**Syllabus Extraction Flow**:
1. Upload syllabus PDF to S3
2. Call Textract AnalyzeDocument API
3. Extract topics and subtopics using table/form detection
4. Parse hierarchical structure
5. Store in DynamoDB

**Topic Weighting Algorithm**:
- If past papers provided:
  - Extract questions using Textract
  - Classify questions by topic using Bedrock
  - Calculate frequency: weight = question_count / total_questions
- If no past papers:
  - Assign equal weights to all topics

**Time Allocation Formula**:
```
time_per_topic = (total_study_hours * topic_weight) / proficiency_score

Where:
- total_study_hours = days_until_exam * daily_study_hours
- topic_weight = 0 to 1 (from past paper analysis)
- proficiency_score = 0 to 100 (current mastery level)

Inverse relationship: lower proficiency = more time allocated
```

**Schedule Generation Prompt**:
```
Generate a study schedule:

Exam date: {exam_date}
Days available: {days}
Daily study time: {hours} hours
Topics: {topics_with_weights}
Current proficiency: {proficiency_data}

Requirements:
1. Allocate more time to weak topics (proficiency < 50)
2. Include review sessions every 7 days
3. Add rest days every 6 days
4. Front-load difficult topics
5. Include buffer time (10% of total)

Return JSON format:
{
  "schedule": [
    {
      "date": "2024-02-15",
      "topics": ["calculus", "algebra"],
      "hours": 3,
      "type": "study"
    },
    {
      "date": "2024-02-16",
      "topics": ["calculus"],
      "hours": 2,
      "type": "review"
    }
  ]
}
```

**Visual Learning Path**:
- Generate Gantt chart showing topic progression
- Color-code by proficiency level (red: weak, yellow: medium, green: strong)
- Show milestones (25%, 50%, 75%, 100% syllabus coverage)
- Render using Chart.js or D3.js

## 5. Data Models

### 5.1 User Profile

**DynamoDB Table**: `Users`

**Schema**:
```json
{
  "PK": "USER#<user_id>",
  "SK": "PROFILE",
  "email": "student@example.com",
  "name": "Student Name",
  "language_preference": "en",
  "created_at": "2024-02-15T10:30:00Z",
  "last_login": "2024-02-15T10:30:00Z",
  "subscription_tier": "free",
  "preferences": {
    "visual_preference_score": 0.6,
    "daily_challenge_count": 5,
    "notification_enabled": true
  }
}
```

**Access Patterns**:
- Get user by ID: Query PK=USER#<user_id>, SK=PROFILE
- Update preferences: UpdateItem on PK=USER#<user_id>, SK=PROFILE

### 5.2 Proficiency Model

**DynamoDB Table**: `Users` (same table, different SK)

**Schema**:
```json
{
  "PK": "USER#<user_id>",
  "SK": "PROFICIENCY#<topic_id>#<timestamp>",
  "proficiency_score": 75,
  "confidence": 0.85,
  "reasoning": "Student demonstrated solid understanding...",
  "topic": "calculus_derivatives",
  "timestamp": "2024-02-15T10:30:00Z",
  "ttl": 1234567890
}
```

**Access Patterns**:
- Get latest proficiency for topic: Query PK=USER#<user_id>, SK begins_with PROFICIENCY#<topic_id>, ScanIndexForward=false, Limit=1
- Get proficiency history: Query PK=USER#<user_id>, SK begins_with PROFICIENCY#<topic_id>

**TTL Policy**: Keep last 10 records per topic, delete older records after 90 days


### 5.3 Session Tracking

**DynamoDB Table**: `Sessions`

**Schema**:
```json
{
  "PK": "SESSION#<session_id>",
  "SK": "METADATA",
  "user_id": "user123",
  "started_at": "2024-02-15T10:30:00Z",
  "ended_at": "2024-02-15T11:00:00Z",
  "duration_seconds": 1800,
  "interactions": [
    {
      "type": "question",
      "timestamp": "2024-02-15T10:31:00Z",
      "topic": "calculus",
      "proficiency_before": 70,
      "proficiency_after": 75
    }
  ],
  "modules_used": ["core_tutor", "visualization"],
  "ttl": 1234567890
}
```

**Access Patterns**:
- Get session by ID: Query PK=SESSION#<session_id>
- Get user sessions: GSI on user_id, Query user_id=<user_id>

**TTL Policy**: Delete sessions after 30 days

### 5.4 Mindmap Storage

**DynamoDB Table**: `Mindmaps`

**Schema**:
```json
{
  "PK": "USER#<user_id>",
  "SK": "MINDMAP#<mindmap_id>",
  "title": "Calculus Concepts",
  "graph": {
    "nodes": [
      {
        "id": "node1",
        "label": "Derivatives",
        "description": "Rate of change",
        "links": ["https://wikipedia.org/...", "CITATION#123"]
      }
    ],
    "edges": [
      {
        "source": "node1",
        "target": "node2",
        "relationship": "prerequisite"
      }
    ]
  },
  "created_at": "2024-02-15T10:30:00Z",
  "updated_at": "2024-02-15T11:00:00Z"
}
```

**Access Patterns**:
- Get user mindmaps: Query PK=USER#<user_id>, SK begins_with MINDMAP#
- Get specific mindmap: Query PK=USER#<user_id>, SK=MINDMAP#<mindmap_id>

### 5.5 Citation Cache

**DynamoDB Table**: `Citations`

**Schema**:
```json
{
  "PK": "CITATION#<citation_id>",
  "SK": "METADATA",
  "title": "Deep Learning for Education",
  "authors": ["Smith, J.", "Doe, A."],
  "year": 2023,
  "citation_count": 145,
  "abstract": "This paper explores...",
  "doi": "10.1234/example",
  "pdf_url": "https://...",
  "bookmarked_by": ["user1", "user2"],
  "search_queries": ["deep learning education", "AI tutoring"],
  "ttl": 1234567890
}
```

**Access Patterns**:
- Get citation by ID: Query PK=CITATION#<citation_id>
- Search citations: GSI on search_queries (for cache lookup)

**TTL Policy**: Delete cached citations after 7 days

## 6. Workflow & Orchestration Design

### 6.1 Step Functions State Machine: Core Tutor Flow

**State Machine Name**: `CoreTutorWorkflow`

**States**:
1. **CheckImageUpload**: Choice state - has image?
   - Yes → OCRProcessing
   - No → DiagnosticAnalysis
2. **OCRProcessing**: Task state - invoke OCR Lambda
3. **DiagnosticAnalysis**: Task state - invoke Diagnostic Lambda
4. **GetProficiency**: Task state - query DynamoDB for proficiency
5. **ParallelExecution**: Parallel state
   - Branch 1: ExplanationGeneration (invoke Explanation Lambda)
   - Branch 2: CheckVisualizationNeed (Choice state)
     - If needed → VisualizationGeneration
     - Else → Skip
6. **AggregateResults**: Task state - combine outputs
7. **UpdateProficiency**: Task state - write to DynamoDB
8. **GenerateXAI**: Task state - invoke XAI Lambda
9. **ReturnResponse**: Task state - format and return

**Error Handling**:
- Each task has retry policy: 3 attempts, exponential backoff
- Catch blocks for all tasks: log to CloudWatch, return user-friendly error

**Execution Time**: Target < 5 seconds for 95% of requests

### 6.2 Step Functions State Machine: Visualization Flow

**State Machine Name**: `VisualizationWorkflow`

**States**:
1. **CheckPreRendered**: Task state - query S3 for pre-rendered template
2. **PreRenderedExists**: Choice state
   - Yes → ReturnCloudFrontURL
   - No → GenerateManim
3. **GenerateManim**: Task state - invoke Manim code generation Lambda
4. **ValidateCode**: Task state - syntax check
5. **SubmitRenderJob**: Task state - send to SQS queue
6. **WaitForRender**: Wait state - poll for completion (max 2 min)
7. **CheckRenderStatus**: Task state - check SNS notification
8. **RenderComplete**: Choice state
   - Success → UploadToS3
   - Failure → RetryOrFail
9. **UploadToS3**: Task state - store video
10. **UpdateCache**: Task state - write to DynamoDB
11. **ReturnCloudFrontURL**: Task state - return URL

**Timeout**: 3 minutes total (2 min rendering + 1 min overhead)

### 6.3 Step Functions State Machine: Exam Planning Flow

**State Machine Name**: `ExamPlanningWorkflow`

**States**:
1. **CollectInputs**: Task state - validate exam date, study time, target
2. **UploadSyllabus**: Task state - upload PDF to S3
3. **ExtractSyllabus**: Task state - invoke Textract
4. **ParseTopics**: Task state - parse Textract output
5. **CheckPastPapers**: Choice state
   - Yes → ExtractPastPapers
   - No → AssignDefaultWeights
6. **ExtractPastPapers**: Task state - invoke Textract
7. **ClassifyQuestions**: Task state - invoke Bedrock for classification
8. **CalculateWeights**: Task state - compute topic weights
9. **GetProficiencyData**: Task state - query DynamoDB
10. **AllocateTime**: Task state - run time allocation algorithm
11. **GenerateSchedule**: Task state - invoke Bedrock for schedule
12. **CreateVisualPath**: Task state - generate Gantt chart
13. **StoreSchedule**: Task state - write to DynamoDB
14. **ReturnPlan**: Task state - return schedule and visual path

**Execution Time**: Target < 30 seconds

## 7. Sequence Flow

### 7.1 Student Asks Question (Text-Based)

1. Student types question in React UI
2. Frontend sends POST request to API Gateway `/ask`
3. API Gateway validates JWT token via Cognito
4. API Gateway invokes Step Functions `CoreTutorWorkflow`
5. Step Functions executes:
   - CheckImageUpload: No image → skip OCR
   - DiagnosticAnalysis: Lambda analyzes question, calls Bedrock
   - GetProficiency: Lambda queries DynamoDB for proficiency score
   - ParallelExecution:
     - ExplanationGeneration: Lambda generates adaptive explanation using Bedrock
     - CheckVisualizationNeed: Based on user preference, may skip
   - AggregateResults: Combine explanation and metadata
   - UpdateProficiency: Lambda writes new proficiency score to DynamoDB
   - GenerateXAI: Lambda generates explanation for why this response was given
6. Step Functions returns response to API Gateway
7. API Gateway returns JSON to frontend
8. Frontend displays explanation and XAI reasoning
9. CloudWatch logs all operations, X-Ray traces request

**Total Time**: ~2-3 seconds


### 7.2 Student Uploads Handwritten Note

1. Student uploads image via React UI
2. Frontend sends POST request to API Gateway `/upload-image`
3. API Gateway validates JWT, returns pre-signed S3 URL
4. Frontend uploads image directly to S3
5. S3 triggers Lambda via event notification
6. Lambda invokes Step Functions `CoreTutorWorkflow`
7. Step Functions executes:
   - CheckImageUpload: Yes → OCRProcessing
   - OCRProcessing: Lambda calls Textract, extracts text
   - DiagnosticAnalysis: Lambda analyzes extracted text
   - (Continue as in 7.1)
8. Response returned to frontend via WebSocket (for real-time updates)

**Total Time**: ~5-7 seconds (OCR adds 3-4 seconds)

### 7.3 Student Requests Visualization

1. Student clicks "Show Visualization" button
2. Frontend sends POST request to API Gateway `/visualize`
3. API Gateway invokes Step Functions `VisualizationWorkflow`
4. Step Functions executes:
   - CheckPreRendered: Lambda queries S3 for template
   - If exists: Return CloudFront URL immediately (~1 second)
   - If not exists:
     - GenerateManim: Lambda calls Bedrock to generate code
     - ValidateCode: Lambda checks syntax
     - SubmitRenderJob: Lambda sends to SQS
     - EC2 Spot Instance picks up job, renders video
     - UploadToS3: EC2 uploads video to S3
     - SNS notification sent to Lambda
     - UpdateCache: Lambda writes to DynamoDB
     - ReturnCloudFrontURL: Return URL
5. Frontend displays video player with CloudFront URL

**Total Time**: 
- Pre-rendered: ~1 second
- Custom render: ~60-120 seconds

### 7.4 Student Generates Exam Plan

1. Student fills exam planning form (date, time, target)
2. Student uploads syllabus PDF and optional past papers
3. Frontend sends POST request to API Gateway `/plan-exam`
4. API Gateway invokes Step Functions `ExamPlanningWorkflow`
5. Step Functions executes:
   - CollectInputs: Validate inputs
   - UploadSyllabus: Store PDF in S3
   - ExtractSyllabus: Textract extracts topics
   - ParseTopics: Lambda parses hierarchical structure
   - CheckPastPapers: If provided, extract and classify
   - CalculateWeights: Compute topic importance
   - GetProficiencyData: Query DynamoDB for current proficiency
   - AllocateTime: Run time allocation algorithm
   - GenerateSchedule: Bedrock generates daily schedule
   - CreateVisualPath: Generate Gantt chart
   - StoreSchedule: Write to DynamoDB
6. Frontend displays schedule and visual learning path

**Total Time**: ~20-30 seconds

## 8. Security & Compliance Design

### 8.1 Authentication & Authorization

**AWS Cognito User Pools**:
- Email/password authentication
- JWT tokens with 1-hour expiration
- Refresh tokens with 30-day expiration
- MFA optional (SMS or TOTP)

**API Gateway Authorization**:
- Cognito authorizer on all protected endpoints
- Token validation before invoking Lambda
- Rate limiting: 100 requests per minute per user

**IAM Roles**:
- Lambda execution role: Minimal permissions (DynamoDB, S3, Bedrock, Textract)
- Step Functions execution role: Invoke Lambda, write CloudWatch logs
- EC2 instance role: Read SQS, write S3, send SNS

### 8.2 Data Encryption

**At Rest**:
- DynamoDB: AWS KMS encryption enabled by default
- S3: Server-side encryption with S3-managed keys (SSE-S3)
- CloudWatch Logs: Encrypted with AWS-managed keys

**In Transit**:
- TLS 1.2+ for all API Gateway endpoints
- HTTPS for CloudFront distributions
- Encrypted connections between AWS services

### 8.3 Data Privacy

**PII Handling**:
- Email stored in Cognito and DynamoDB (encrypted)
- No PII in CloudWatch logs (sanitize before logging)
- Student responses and proficiency data encrypted at rest

**Data Retention**:
- Session data: 30 days (TTL in DynamoDB)
- Proficiency history: 90 days (TTL in DynamoDB)
- Uploaded images: 30 days (S3 lifecycle policy)
- Videos: Indefinite (user can delete)

**Data Deletion**:
- User requests deletion via UI
- Lambda function deletes:
  - Cognito user account
  - All DynamoDB records (user profile, proficiency, sessions)
  - All S3 objects (images, videos)
- Deletion completes within 24 hours

### 8.4 Compliance

**Indian Data Protection**:
- Data stored in AWS Mumbai region (ap-south-1)
- Compliant with Digital Personal Data Protection Act (DPDPA) 2023
- User consent collected during registration
- Privacy policy clearly states data usage

**AWS Compliance**:
- SOC 2 Type II certified
- ISO 27001 certified
- GDPR compliant (for future international expansion)

## 9. Scalability & Reliability Strategy

### 9.1 Auto-Scaling

**Lambda**:
- Concurrent execution limit: 1000 (default)
- Reserved concurrency for critical functions (Diagnostic, Explanation)
- Provisioned concurrency for low-latency requirements (optional)

**DynamoDB**:
- On-demand capacity mode (auto-scales with traffic)
- No manual capacity planning required
- Handles sudden traffic spikes automatically

**EC2 Spot Instances**:
- Auto Scaling Group with target tracking policy
- Scale based on SQS queue depth
- Mix of instance types for availability (t3.medium, t3.large)

### 9.2 High Availability

**Multi-AZ Deployment**:
- DynamoDB: Automatically replicated across 3 AZs
- Lambda: Runs in multiple AZs by default
- S3: 99.999999999% durability, replicated across AZs

**CloudFront**:
- Global edge network (200+ locations)
- Automatic failover to origin if edge cache fails
- 99.9% uptime SLA

**Failure Handling**:
- Lambda retries: 3 attempts with exponential backoff
- Step Functions retries: Configurable per task
- Dead Letter Queues (DLQ) for failed messages
- CloudWatch alarms for error rate thresholds

### 9.3 Performance Optimization

**Caching Strategy**:
- Bedrock prompt caching: 60%+ cache hit rate
- CloudFront caching: 24-hour TTL for videos
- DynamoDB caching: Application-level cache for proficiency data (5-minute TTL)
- Citation caching: 7-day TTL in DynamoDB

**Latency Reduction**:
- CloudFront edge locations for global low-latency delivery
- DynamoDB single-digit millisecond latency
- Lambda in same region as DynamoDB (ap-south-1)
- Parallel execution in Step Functions where possible

**Batch Processing**:
- Use Bedrock Batch API for non-real-time requests (50% cost savings)
- Batch proficiency updates (write to DynamoDB every 5 interactions)

## 10. Cost Optimization Strategy

### 10.1 Demo Phase Optimization ($26-32/month)

**Bedrock**:
- Use Claude 3 Haiku instead of Sonnet (83% cheaper)
- Implement aggressive prompt caching (60% hit rate)
- Estimated: $10-12/month for 5K queries

**Textract**:
- Use only for handwritten text (not printed)
- Estimated: $0.75/month for 500 pages

**Lambda**:
- Stay within 1M free tier invocations
- Optimize memory allocation (512MB for most functions)
- Estimated: $0 (free tier)

**EC2**:
- Use Spot Instances (70% cheaper)
- Pre-render 50 common topics (95% reduction in rendering)
- Estimated: $0.10-0.50/month for 10 hours

**S3 + CloudFront**:
- Compress videos (50% size reduction)
- Aggressive CloudFront caching (reduce S3 GET requests)
- Estimated: $4.50/month

**Total**: $26-32/month


### 10.2 Scaled Phase Optimization ($133-157/month for 100 users)

**Bedrock**:
- Upgrade to Claude 3.5 Sonnet for better quality
- Use Batch API for 50% cost reduction
- Estimated: $60-75/month for 40K queries

**Lambda**:
- Beyond free tier: $2/month for 150K invocations
- Optimize cold starts with provisioned concurrency (optional)

**EC2**:
- More rendering jobs, but still use Spot Instances
- Estimated: $15-25/month for 50 hours

**S3 + CloudFront**:
- More video storage and delivery
- Estimated: $36/month

**Total**: $133-157/month

### 10.3 Cost Monitoring

**CloudWatch Billing Alarms**:
- Alert if monthly cost exceeds $35 (demo), $70 (MVP), $160 (scaled)
- Daily cost tracking dashboard

**Cost Allocation Tags**:
- Tag all resources by module (core_tutor, depth_tools, visual, planning)
- Identify cost-heavy modules for optimization

**Cost Optimization Recommendations**:
- Review CloudWatch Insights for unused resources
- Analyze Lambda memory usage, right-size allocations
- Monitor DynamoDB read/write patterns, optimize queries

## 11. Observability & Monitoring

### 11.1 CloudWatch Logging

**Log Groups**:
- `/aws/lambda/diagnostic-agent`
- `/aws/lambda/explanation-agent`
- `/aws/lambda/ocr-agent`
- `/aws/lambda/citation-researcher`
- `/aws/lambda/mindmap-generator`
- `/aws/lambda/coding-agent`
- `/aws/lambda/web-search-agent`
- `/aws/lambda/visualization-engine`
- `/aws/lambda/challenge-generator`
- `/aws/lambda/exam-planner`
- `/aws/lambda/xai-module`
- `/aws/stepfunctions/core-tutor-workflow`
- `/aws/stepfunctions/visualization-workflow`
- `/aws/stepfunctions/exam-planning-workflow`

**Log Retention**: 7 days (demo), 30 days (production)

**Structured Logging**:
```json
{
  "timestamp": "2024-02-15T10:30:00Z",
  "level": "INFO",
  "agent": "diagnostic",
  "user_id": "user123",
  "session_id": "session456",
  "action": "analyze_response",
  "proficiency_score": 75,
  "duration_ms": 1234,
  "bedrock_tokens": 500
}
```

### 11.2 CloudWatch Metrics

**Custom Metrics**:
- `AgentInvocations` (by agent type)
- `AgentDuration` (by agent type)
- `BedrockTokenUsage` (input/output tokens)
- `TextractPageCount`
- `ProficiencyScoreChange` (delta per interaction)
- `VisualizationRequests` (pre-rendered vs custom)
- `CacheHitRate` (Bedrock, CloudFront, DynamoDB)

**Metric Alarms**:
- `HighErrorRate`: Error rate > 5% for 5 minutes
- `HighLatency`: P95 latency > 5 seconds for 5 minutes
- `LowCacheHitRate`: Cache hit rate < 40% for 1 hour
- `HighCost`: Daily cost > $2 (demo), $5 (MVP), $10 (scaled)

### 11.3 X-Ray Tracing

**Trace All Requests**:
- API Gateway → Step Functions → Lambda → Bedrock/Textract → DynamoDB
- Visualize service map and identify bottlenecks
- Analyze latency breakdown by service

**Trace Annotations**:
- `user_id`: For filtering traces by user
- `agent_type`: For filtering by agent
- `proficiency_score`: For correlating performance with proficiency

**Trace Sampling**:
- 100% sampling for demo phase
- 10% sampling for production (reduce costs)

### 11.4 Dashboards

**Operational Dashboard**:
- Request count (last 24 hours)
- Error rate (last 24 hours)
- P50, P95, P99 latency
- Active users (last 24 hours)
- Cost (current month)

**Agent Performance Dashboard**:
- Invocations per agent
- Average duration per agent
- Error rate per agent
- Bedrock token usage per agent

**User Engagement Dashboard**:
- Daily active users
- Average session duration
- Module usage distribution
- Visualization preference (visual vs text)

## 12. Production Readiness

### 12.1 Infrastructure as Code

**AWS CDK (TypeScript)**:
- Define all infrastructure in code
- Version control in Git
- Automated deployment via CI/CD

**Stack Structure**:
- `NetworkStack`: VPC, subnets (if needed for EC2)
- `AuthStack`: Cognito User Pool
- `DataStack`: DynamoDB tables
- `StorageStack`: S3 buckets, CloudFront distributions
- `ComputeStack`: Lambda functions, EC2 Auto Scaling Group
- `OrchestrationStack`: Step Functions state machines
- `APIStack`: API Gateway
- `MonitoringStack`: CloudWatch dashboards, alarms

### 12.2 CI/CD Pipeline

**AWS Amplify**:
- Frontend deployment (React app)
- Automatic builds on Git push
- Preview environments for pull requests

**GitHub Actions** (or AWS CodePipeline):
- Backend deployment (Lambda, Step Functions)
- Run unit tests before deployment
- Deploy to staging, then production

**Deployment Stages**:
1. **Dev**: Automatic deployment on push to `dev` branch
2. **Staging**: Automatic deployment on push to `staging` branch
3. **Production**: Manual approval required, deploy from `main` branch

### 12.3 Testing Strategy

**Unit Tests**:
- Test individual Lambda functions with mock inputs
- Test Bedrock prompt templates
- Test DynamoDB queries
- Coverage target: 80%+

**Integration Tests**:
- Test Step Functions workflows end-to-end
- Test API Gateway endpoints
- Test authentication flow

**Load Tests**:
- Simulate 50 concurrent users (demo), 100 users (production)
- Verify response times under load
- Identify bottlenecks

**Security Tests**:
- Penetration testing on API Gateway
- Verify JWT token validation
- Test data encryption

### 12.4 Disaster Recovery

**Backup Strategy**:
- DynamoDB: Point-in-time recovery enabled (35-day retention)
- S3: Versioning enabled for critical buckets
- Lambda: Code stored in Git, redeployable anytime

**Recovery Time Objective (RTO)**: 1 hour
**Recovery Point Objective (RPO)**: 1 hour

**Disaster Scenarios**:
1. **Region Failure**: Redeploy to different region (manual, 1 hour)
2. **Data Corruption**: Restore DynamoDB from point-in-time backup (30 minutes)
3. **Lambda Failure**: Automatic retry by Step Functions (seconds)
4. **S3 Failure**: AWS handles automatically (transparent to users)

## 13. Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

Before defining the correctness properties, I need to analyze each acceptance criterion from the requirements document to determine which are testable as properties, examples, or edge cases.


### Property Reflection

After analyzing all acceptance criteria, I've identified several areas where properties can be consolidated to eliminate redundancy:

**Consolidation Opportunities:**

1. **Proficiency Score Range Validation** (1.1): This property validates that all proficiency scores are in [0, 100]. This subsumes any individual checks for valid ranges.

2. **Adaptive Explanation Complexity** (2.2, 2.3, 2.4): These three properties test the same behavior (adaptive complexity) across different proficiency ranges. They can be combined into a single comprehensive property that tests adaptation across all ranges.

3. **Performance Requirements**: Multiple properties test response time limits (1.3, 2.5, 3.2, 4.5, 5.1, 5.6, 6.3, 7.2, 8.4, 10.6, 11.2, 12.2, 12.5, 14.1, 14.3, 14.4). These can be grouped into a single property that tests performance across all operations.

4. **Data Persistence** (1.2, 5.5, 6.6, 10.1, 12.5): Multiple properties test that data is correctly stored in DynamoDB. These can be combined into a single property about data persistence correctness.

5. **Metadata Completeness** (5.2, 7.3, 17.1, 17.2): Multiple properties test that required fields are present in outputs. These can be combined into a single property about output completeness.

6. **OCR Accuracy** (3.3, 3.4): These test accuracy for different text types. They can be combined into a single property that tests OCR accuracy across all text types.

**Properties to Keep Separate:**

- Properties testing distinct behaviors (e.g., proficiency initialization vs proficiency updates)
- Properties testing different algorithms (e.g., spaced repetition vs difficulty distribution)
- Properties testing different system components (e.g., authentication vs authorization)
- Properties with unique validation logic (e.g., mindmap graph structure vs citation search)

After reflection, I've reduced 90+ testable criteria to approximately 40 unique, non-redundant properties that provide comprehensive validation coverage.

### Correctness Properties


#### Property 1: Proficiency Score Validity

*For any* student response analyzed by the Diagnostic_Agent, the calculated Proficiency_Level score should be a number between 0 and 100 inclusive.

**Validates: Requirements 1.1**

#### Property 2: Proficiency Persistence

*For any* calculated Proficiency_Level, when stored in the Proficiency_Model, the record should contain the score, timestamp, topic metadata, and be retrievable by user ID and topic.

**Validates: Requirements 1.2**

#### Property 3: Proficiency Initialization

*For any* topic that a student has not previously studied, the initial Proficiency_Level should be set to 50.

**Validates: Requirements 1.4**

#### Property 4: Proficiency History Retention

*For any* topic with proficiency updates, the system should maintain at least the last 10 proficiency scores in chronological order.

**Validates: Requirements 1.5**

#### Property 5: Adaptive Explanation Complexity

*For any* explanation generation request, the explanation complexity (measured by vocabulary level and depth) should correlate with the student's Proficiency_Level: simplified for scores below 40, intermediate for scores 40-70, and advanced for scores above 70.

**Validates: Requirements 2.2, 2.3, 2.4**

#### Property 6: Proficiency Retrieval Before Explanation

*For any* explanation generation request, the system should retrieve the student's current Proficiency_Level for the relevant topic before generating the explanation.

**Validates: Requirements 2.1**

#### Property 7: Image Format Acceptance

*For any* image upload, the system should accept files in JPEG, PNG, or PDF format and reject all other formats with a clear error message.

**Validates: Requirements 3.1**

#### Property 8: OCR Accuracy Thresholds

*For any* text extraction from images, the OCR_Agent should achieve at least 90% accuracy on printed text and at least 75% accuracy on handwritten text, measured against ground truth.

**Validates: Requirements 3.3, 3.4**

#### Property 9: Low Confidence Handling

*For any* OCR extraction with average confidence below 60%, the system should request the student to re-upload the image or manually enter the text rather than proceeding with low-quality extraction.

**Validates: Requirements 3.5**

#### Property 10: OCR to Diagnostic Handoff

*For any* successful text extraction, the extracted text should be passed to the Diagnostic_Agent for analysis without requiring additional student action.

**Validates: Requirements 3.6**

#### Property 11: XAI Explanation Presence

*For any* hint or learning path recommendation provided by the system, an XAI explanation should be generated that references the student's Proficiency_Level and learning context.

**Validates: Requirements 4.1, 4.2**

#### Property 12: XAI Detailed Reasoning

*For any* request for more detail on a recommendation, the XAI_Module should provide reasoning that includes specific proficiency scores and identified learning patterns.

**Validates: Requirements 4.4**

#### Property 13: Citation Result Count and Relevance

*For any* citation search request, the Citation_Researcher should return at least 5 results that are relevant to the search topic.

**Validates: Requirements 5.1**

#### Property 14: Citation Metadata Completeness

*For any* citation returned by the system, the citation should include title, authors, publication year, citation count, and an abstract preview of at least 200 characters.

**Validates: Requirements 5.2, 5.3**

#### Property 15: Citation Quick Actions Availability

*For any* displayed citation, the system should provide quick actions including PDF link, DOI link, and bookmark functionality.

**Validates: Requirements 5.4**

#### Property 16: Bookmark Persistence

*For any* citation bookmarked by a student, the citation should be stored in DynamoDB associated with the student's profile and retrievable in future sessions.

**Validates: Requirements 5.5**

#### Property 17: Mindmap Graph Structure Validity

*For any* generated mindmap, the graph structure should be valid with nodes representing concepts and edges representing relationships, and should be serializable to JSON format compatible with D3.js.

**Validates: Requirements 6.2**

#### Property 18: Mindmap Node Resource Attachment

*For any* mindmap node, the system should attach at least one relevant link (Wikipedia, citation, or educational resource) to provide additional learning context.

**Validates: Requirements 6.4**

#### Property 19: Mindmap Persistence

*For any* created mindmap, the graph structure should be stored in DynamoDB and retrievable by the student in future sessions.

**Validates: Requirements 6.6**

#### Property 20: Code Execution Sandbox Isolation

*For any* student code submitted for execution, the code should run in a secure Lambda sandbox with no network access, restricted file system access, and no access to AWS credentials.

**Validates: Requirements 7.1**

#### Property 21: Test Failure Message Completeness

*For any* failed test execution, the system should provide a failure message that includes the test case, expected output, actual output, and error message (if applicable).

**Validates: Requirements 7.3**

#### Property 22: Fix Suggestion Generation

*For any* failed test execution, the system should use AWS Bedrock to generate specific fix suggestions adapted to the student's programming proficiency level.

**Validates: Requirements 7.4, 7.6**

#### Property 23: Runtime Error Capture

*For any* code execution that encounters a runtime error, the system should capture and return the error message and stack trace to help the student debug.

**Validates: Requirements 7.5**

#### Property 24: Contextual Search Filtering

*For any* web search with an available mindmap context, the search results should be filtered to match concepts present in the mindmap, improving relevance.

**Validates: Requirements 8.2**

#### Property 25: Search Result Synthesis

*For any* set of search results, the system should use AWS Bedrock to synthesize key information and cite sources for each synthesized point.

**Validates: Requirements 8.5, 8.6**

#### Property 26: Visualization Choice Presentation

*For any* visualization request, the system should offer the student a choice between visual animation and text explanation before proceeding.

**Validates: Requirements 9.1**

#### Property 27: Manim Code Generation

*For any* visual animation request, the Visualization_Engine should generate valid Manim Python code that defines a clear learning goal.

**Validates: Requirements 9.2, 9.3**

#### Property 28: Animation Rendering Infrastructure

*For any* animation rendering job, the system should use EC2 Spot Instances to execute Manim and generate video output, optimizing cost.

**Validates: Requirements 9.4**

#### Property 29: Visualization Preference Tracking

*For any* animation viewed by a student, the system should track the preference in DynamoDB and use this data to inform future content delivery decisions.

**Validates: Requirements 9.6**

#### Property 30: Proactive Visualization Suggestions

*For any* student who has viewed 3 or more animations, the system should analyze preference patterns and proactively suggest visualizations for suitable topics.

**Validates: Requirements 9.7**

#### Property 31: Challenge Difficulty Distribution

*For any* set of daily challenges generated, the difficulty distribution should be 60% at current proficiency level, 30% slightly harder, and 10% significantly harder.

**Validates: Requirements 10.3**

#### Property 32: Proficiency Update on Challenge Completion

*For any* completed challenge, the system should update the Proficiency_Model based on the student's performance (correct/incorrect answers).

**Validates: Requirements 10.4**

#### Property 33: Spaced Repetition for Weak Topics

*For any* student with weak topics (Proficiency_Level below 50), the Challenge_Generator should schedule review of those topics using spaced repetition intervals (1, 3, 7, 14, 30 days).

**Validates: Requirements 10.5**

#### Property 34: Topic Weight Calculation

*For any* exam planning with past papers, the system should calculate topic weights based on question frequency, where weight equals question_count divided by total_questions.

**Validates: Requirements 11.4**

#### Property 35: Default Weight Assignment

*For any* exam planning without past papers, the system should assign equal weights to all syllabus topics.

**Validates: Requirements 11.5**

#### Property 36: Time Allocation Formula

*For any* exam schedule generation, study time per topic should be allocated proportional to topic weight and inversely proportional to current Proficiency_Level, such that weaker topics receive more time.

**Validates: Requirements 11.6**

#### Property 37: Schedule Completeness

*For any* generated exam schedule, the schedule should include daily study slots, review sessions for weak topics (proficiency < 50), and rest days (every 6 days).

**Validates: Requirements 11.7**

#### Property 38: Authentication Account Creation

*For any* student registration, the system should create a secure user account in AWS Cognito with encrypted credentials.

**Validates: Requirements 12.1**

#### Property 39: Profile Retrieval Completeness

*For any* successful authentication, the system should retrieve the complete student profile including proficiency data, preferences, and bookmarks from DynamoDB.

**Validates: Requirements 12.3**

#### Property 40: Cross-Device Data Synchronization

*For any* student accessing the system from a new device, all learning data (proficiency, preferences, bookmarks, mindmaps) should be synchronized from DynamoDB.

**Validates: Requirements 12.6**

#### Property 41: Workflow Orchestration via Step Functions

*For any* learning interaction initiated by a student, the system should use AWS Step Functions to orchestrate the workflow and route requests to appropriate agents.

**Validates: Requirements 13.1, 13.2**

#### Property 42: Parallel Agent Execution

*For any* workflow requiring multiple independent agents, Step Functions should execute those agents in parallel to minimize total latency.

**Validates: Requirements 13.3**

#### Property 43: Agent Retry with Exponential Backoff

*For any* agent failure, Step Functions should implement retry logic with exponential backoff for up to 3 attempts before failing.

**Validates: Requirements 13.4**

#### Property 44: Error Logging and User-Friendly Messages

*For any* workflow failure after all retries, the system should log the failure to CloudWatch with full context and return a user-friendly error message to the student.

**Validates: Requirements 13.5**

#### Property 45: Lambda Auto-Scaling

*For any* increase in concurrent users, the system should automatically scale Lambda functions to handle the load without manual intervention.

**Validates: Requirements 14.2**

#### Property 46: Data Encryption at Rest

*For any* student data stored in DynamoDB, the data should be encrypted at rest using AWS KMS.

**Validates: Requirements 15.1**

#### Property 47: TLS for Data in Transit

*For any* API communication, the system should use TLS 1.2 or higher to encrypt data in transit.

**Validates: Requirements 15.2**

#### Property 48: Authentication Token Verification

*For any* request to access student data, the system should verify the authentication token via AWS Cognito before allowing access.

**Validates: Requirements 15.3**

#### Property 49: Data Deletion Completeness

*For any* student data deletion request, the system should remove all personal data from DynamoDB and S3 within 24 hours.

**Validates: Requirements 15.4**

#### Property 50: PII Sanitization in Logs

*For any* event logged to CloudWatch, the log entry should not contain personally identifiable information (email, name, etc.).

**Validates: Requirements 15.5**

#### Property 51: Prompt Caching Effectiveness

*For any* set of Bedrock requests over a 24-hour period, the cache hit rate should be at least 60% to optimize costs.

**Validates: Requirements 16.1**

#### Property 52: Pre-rendered Template Prioritization

*For any* visualization request for a common topic (top 50 topics), the system should use a pre-rendered template from S3 rather than triggering custom rendering.

**Validates: Requirements 16.2**

#### Property 53: Video Compression Ratio

*For any* rendered video, the compressed video file size should be at least 50% smaller than the uncompressed format while maintaining acceptable quality.

**Validates: Requirements 16.5**

#### Property 54: Agent Invocation Logging

*For any* agent execution, the system should log to CloudWatch with timestamp, agent type, execution duration, and outcome (success/failure).

**Validates: Requirements 17.1**

#### Property 55: Error Logging with Context

*For any* error that occurs, the system should log error details including stack trace, error message, and execution context to CloudWatch.

**Validates: Requirements 17.2**

#### Property 56: X-Ray Request Tracing

*For any* request processed by the system, AWS X-Ray should create a trace that spans all services involved (API Gateway, Step Functions, Lambda, Bedrock, DynamoDB).

**Validates: Requirements 17.3**

#### Property 57: Proficiency-Only Adaptation

*For any* content difficulty adaptation, the system should base adjustments solely on demonstrated proficiency scores and not use demographic factors (age, gender, location).

**Validates: Requirements 18.2**

#### Property 58: Bias Incident Logging

*For any* content flagged as potentially biased (by automated detection or user report), the system should log the incident to CloudWatch for review.

**Validates: Requirements 18.4**


## 14. Error Handling

### 14.1 Error Categories

**User Errors**:
- Invalid input format (wrong image type, malformed data)
- Authentication failures (wrong password, expired token)
- Authorization failures (accessing another user's data)
- Response: User-friendly error message, HTTP 400/401/403

**System Errors**:
- AWS service failures (Bedrock timeout, Textract error)
- Database errors (DynamoDB throttling, connection timeout)
- Network errors (API Gateway timeout, Lambda cold start)
- Response: Retry with exponential backoff, fallback to cached data if available, HTTP 500/503

**Business Logic Errors**:
- Proficiency calculation failure (invalid response format)
- Mindmap generation failure (no concepts extracted)
- Schedule generation failure (invalid exam date)
- Response: Log error, return partial result or default value, HTTP 200 with error field

### 14.2 Error Handling Strategies

**Retry Logic**:
- Lambda functions: 3 retries with exponential backoff (1s, 2s, 4s)
- Step Functions: Configurable retry per task
- Bedrock API: 3 retries with jitter to avoid thundering herd
- DynamoDB: Automatic retries with exponential backoff (AWS SDK default)

**Fallback Mechanisms**:
- If Bedrock fails: Use cached response if available, otherwise return generic explanation
- If Textract fails: Ask user to manually enter text
- If proficiency retrieval fails: Use default proficiency of 50
- If visualization rendering fails: Offer text explanation instead

**Circuit Breaker**:
- If Bedrock error rate > 50% for 5 minutes: Switch to cached responses only
- If DynamoDB error rate > 50% for 5 minutes: Return read-only mode
- Automatic recovery when error rate drops below 10%

**Dead Letter Queues**:
- Failed SQS messages (rendering jobs) sent to DLQ
- Manual review and retry of DLQ messages
- CloudWatch alarm when DLQ depth > 10

### 14.3 Error Logging

**Structured Error Logs**:
```json
{
  "timestamp": "2024-02-15T10:30:00Z",
  "level": "ERROR",
  "error_type": "BedrockTimeout",
  "error_message": "Request timed out after 30s",
  "stack_trace": "...",
  "context": {
    "user_id": "user123",
    "session_id": "session456",
    "agent": "explanation",
    "proficiency_score": 75
  },
  "retry_count": 2,
  "will_retry": true
}
```

**Error Metrics**:
- Error count by type (user, system, business logic)
- Error rate by agent
- Mean time to recovery (MTTR)
- Error impact (number of users affected)

## 15. Testing Strategy

### 15.1 Unit Testing

**Lambda Functions**:
- Test each agent function with mock inputs
- Test proficiency calculation logic
- Test adaptive explanation generation
- Test OCR text extraction parsing
- Test mindmap graph generation
- Test schedule generation algorithm
- Coverage target: 80%+

**Test Framework**: pytest (Python), Jest (TypeScript/JavaScript)

**Example Unit Test**:
```python
def test_diagnostic_agent_calculates_valid_proficiency():
    # Arrange
    response = "The derivative of x^2 is 2x"
    question = "What is the derivative of x^2?"
    
    # Act
    result = diagnostic_agent.analyze_response(response, question)
    
    # Assert
    assert 0 <= result.proficiency_score <= 100
    assert result.confidence > 0
    assert result.reasoning is not None
```

### 15.2 Property-Based Testing

**Property Test Framework**: Hypothesis (Python), fast-check (TypeScript)

**Test Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with feature name and property number
- Tag format: `# Feature: ai-adaptive-learning-platform, Property 1: Proficiency Score Validity`

**Example Property Test**:
```python
from hypothesis import given, strategies as st

@given(st.text(min_size=10, max_size=1000))
def test_proficiency_score_always_in_valid_range(student_response):
    """
    Feature: ai-adaptive-learning-platform
    Property 1: Proficiency Score Validity
    
    For any student response, proficiency score should be 0-100
    """
    # Arrange
    question = "Explain calculus"
    
    # Act
    result = diagnostic_agent.analyze_response(student_response, question)
    
    # Assert
    assert 0 <= result.proficiency_score <= 100
```

**Properties to Test**:
- All 58 correctness properties defined in section 13
- Focus on properties that validate universal behaviors
- Use generators to create random but valid test data

### 15.3 Integration Testing

**Step Functions Workflows**:
- Test CoreTutorWorkflow end-to-end
- Test VisualizationWorkflow end-to-end
- Test ExamPlanningWorkflow end-to-end
- Verify correct agent invocation and result aggregation

**API Gateway Endpoints**:
- Test all REST endpoints with valid/invalid inputs
- Test authentication and authorization
- Test rate limiting

**DynamoDB Operations**:
- Test CRUD operations for all tables
- Test query patterns and indexes
- Test TTL expiration

**Test Environment**: Separate AWS account or isolated resources with test data

### 15.4 Performance Testing

**Load Testing**:
- Tool: Apache JMeter or Locust
- Scenarios:
  - 50 concurrent users (demo phase)
  - 100 concurrent users (production phase)
  - Spike test: 0 to 100 users in 1 minute
- Metrics: Response time (P50, P95, P99), error rate, throughput

**Stress Testing**:
- Gradually increase load until system breaks
- Identify bottlenecks (Lambda concurrency, DynamoDB throughput)
- Verify auto-scaling works correctly

**Endurance Testing**:
- Run at 50% capacity for 24 hours
- Monitor for memory leaks, connection leaks
- Verify system remains stable

### 15.5 Security Testing

**Authentication Testing**:
- Test with invalid JWT tokens
- Test with expired tokens
- Test token refresh flow

**Authorization Testing**:
- Test accessing another user's data (should fail)
- Test accessing data without authentication (should fail)

**Penetration Testing**:
- SQL injection attempts (not applicable, using DynamoDB)
- XSS attempts in user inputs
- CSRF attempts on API endpoints

**Data Encryption Testing**:
- Verify TLS 1.2+ for all API calls
- Verify KMS encryption for DynamoDB
- Verify S3 encryption for uploaded files

### 15.6 Usability Testing

**User Testing Sessions**:
- Recruit 10 students (5 high school, 5 college)
- Test all 4 modules with real tasks
- Collect feedback on UI/UX, clarity, usefulness

**Metrics**:
- Task completion rate
- Time to complete tasks
- User satisfaction score (1-10)
- Net Promoter Score (NPS)

**Feedback Collection**:
- Post-session survey
- Screen recording analysis
- Think-aloud protocol during testing

## 16. Post-Hackathon Roadmap

### Phase 1: MVP Enhancements (Weeks 1-8)

**Core Tutor Improvements**:
- Add support for more subjects (physics, chemistry, biology)
- Improve proficiency model with more granular topic tracking
- Add voice input for questions (AWS Transcribe)

**Depth Tools Expansion**:
- Integrate with more academic databases (IEEE, ACM)
- Add collaborative mindmaps (multiple students)
- Improve code execution with more languages (C++, Rust)

**Visual & Challenge Enhancements**:
- Expand pre-rendered template library to 100 topics
- Add interactive animations (student can control playback)
- Implement leaderboards for daily challenges

**Planning Features**:
- Add progress tracking against exam schedule
- Send reminders for study sessions (SNS + email)
- Integrate with calendar apps (Google Calendar, Outlook)

### Phase 2: Scale and Optimize (Weeks 9-16)

**Performance Optimization**:
- Implement Redis caching layer (ElastiCache)
- Optimize DynamoDB indexes for common queries
- Reduce Lambda cold starts with provisioned concurrency

**Cost Optimization**:
- Migrate to Bedrock Batch API for all non-real-time requests
- Implement intelligent pre-rendering based on usage patterns
- Optimize video compression further (target 70% reduction)

**Monitoring Enhancements**:
- Add custom CloudWatch dashboards for business metrics
- Implement anomaly detection for error rates
- Add user behavior analytics (AWS Pinpoint)

### Phase 3: Advanced Features (Weeks 17-24)

**Multi-Language Support**:
- Add Hindi, Telugu, Tamil, Bengali interfaces
- Translate explanations to regional languages
- Support code-switching (mixing English and Hindi)

**Collaborative Learning**:
- Add study groups (multiple students in same session)
- Peer-to-peer explanation sharing
- Group challenges and competitions

**Advanced AI Features**:
- Emotion detection from text (identify frustration, confusion)
- Adaptive pacing (slow down if student is struggling)
- Predictive analytics (predict exam performance)

**Mobile App**:
- Native iOS and Android apps
- Offline mode for downloaded content
- Push notifications for reminders

### Phase 4: Enterprise and Scale (Weeks 25+)

**B2B Features**:
- Teacher dashboard for monitoring student progress
- School/college admin panel
- Bulk user management and reporting

**Advanced Analytics**:
- Learning analytics dashboard for educators
- Cohort analysis and comparison
- Intervention recommendations for struggling students

**Compliance and Certification**:
- GDPR compliance for international expansion
- SOC 2 Type II certification
- Accessibility certification (WCAG 2.1 Level AA)

**Global Expansion**:
- Multi-region deployment (US, Europe, Asia)
- Localization for 10+ languages
- Partnership with educational institutions

## 17. Conclusion

The AI-Powered Adaptive Learning Platform represents a comprehensive, production-ready solution for personalized education at scale. By leveraging 100% AWS-native services, the platform achieves:

- **Adaptive Intelligence**: Real-time proficiency tracking ensures every interaction is personalized
- **Explainability**: XAI module builds trust through transparent reasoning
- **Multi-Modal Learning**: Seamless integration of text, visual, code, and research tools
- **Cost Efficiency**: Operates at $26-32/month for demo phase through aggressive optimization
- **Production Readiness**: Enterprise-grade architecture with monitoring, security, and auto-scaling
- **Hackathon Alignment**: Maximizes scoring across all evaluation criteria (problem relevance, social impact, innovation, AWS-native usage, feasibility, demo readiness, scalability)

The platform addresses critical gaps in the Indian education market while demonstrating technical excellence and scalability. With a clear roadmap for post-hackathon development, the system is positioned for long-term success and impact.
