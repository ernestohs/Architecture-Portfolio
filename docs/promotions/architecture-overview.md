# Promotions System Architecture Overview

## Core Features Explained
### Campaign and Promotion Management
The system allows marketing teams to create promotional campaigns with multiple promotions. Each promotion can have different rules, target segments, and redemption methods. Campaigns are managed through the Admin API, which is built with Node.js running on Lambda.
### Rules Engine
A flexible rules engine (built with Python) evaluates promotion eligibility based on customer attributes, purchase history, location, and other parameters. Rules are stored as JSON configurations in DynamoDB and processed by the Validation Service.
### Multi-channel Distribution
Promotions can be distributed through various channels: email, mobile app, website, in-store, partner networks. Each channel integrates with the Promotion API to access relevant offers.
### Real-time Redemption Processing
When customers redeem promotions, the Redemption Service validates the transaction, updates inventory, and records analytics data. SQS queues ensure reliable processing of high-volume redemptions.
### Analytics and Reporting
The Analytics Service aggregates redemption data to provide insights on campaign performance, customer behavior, and ROI. This service uses Python for data processing and feeds dashboards in the admin portal.

## Technical Implementation Highlights

1. Serverless Architecture: Using Lambda functions for all backend services provides automatic scaling during high-traffic promotional periods.
2. Data Storage Strategy:
    - DynamoDB for transactional data (campaigns, promotions, redemptions)
    - ElasticSearch for advanced promotion search capabilities
    - S3 for promotional assets (images, videos, PDFs)
3. Event-driven Processing:
    - SQS queues decouple services for reliability
    - Step Functions orchestrate complex workflows like multi-stage promotions
4. Security Model:
    - Cognito manages authentication
    - Fine-grained IAM roles for each Lambda function
    - Field-level encryption for sensitive promotion data

## System Components and Flow

```mermaid
flowchart TB
    subgraph "Frontend"
        React[React Web App]
        Mobile[Mobile Apps]
    end
    
    subgraph "API Gateway & Authentication"
        APIG[API Gateway]
        Cognito[Cognito User Pool]
    end
    
    subgraph "Backend Services"
        PromotionAPI[Promotion API\nLambda + Node.js]
        AdminAPI[Admin API\nLambda + Node.js]
        ValidationService[Validation Service\nLambda + Python]
        RedemptionService[Redemption Service\nLambda + Python]
        AnalyticsService[Analytics Service\nLambda + Python]
    end
    
    subgraph "Processing"
        SQS[SQS Queues]
        StepFunctions[Step Functions]
    end
    
    subgraph "Data Layer"
        DynamoDB[(DynamoDB)]
        S3[(S3 for assets)]
        ElasticSearch[(ElasticSearch\nfor search)]
    end
    
    subgraph "Monitoring & Observability"
        CloudWatch[CloudWatch]
        XRay[X-Ray]
    end
    
    React --> APIG
    Mobile --> APIG
    APIG --> Cognito
    Cognito --> APIG
    
    APIG --> PromotionAPI
    APIG --> AdminAPI
    APIG --> ValidationService
    APIG --> RedemptionService
    
    PromotionAPI --> SQS
    AdminAPI --> SQS
    ValidationService --> SQS
    RedemptionService --> SQS
    
    SQS --> StepFunctions
    StepFunctions --> AnalyticsService
    
    PromotionAPI --> DynamoDB
    AdminAPI --> DynamoDB
    ValidationService --> DynamoDB
    RedemptionService --> DynamoDB
    AnalyticsService --> DynamoDB
    
    PromotionAPI --> S3
    AdminAPI --> S3
    
    PromotionAPI --> ElasticSearch
    AdminAPI --> ElasticSearch
    
    PromotionAPI -.-> CloudWatch
    AdminAPI -.-> CloudWatch
    ValidationService -.-> CloudWatch
    RedemptionService -.-> CloudWatch
    AnalyticsService -.-> CloudWatch
    SQS -.-> CloudWatch
    StepFunctions -.-> CloudWatch
    
    PromotionAPI -.-> XRay
    AdminAPI -.-> XRay
    ValidationService -.-> XRay
    RedemptionService -.-> XRay
    AnalyticsService -.-> XRay
```

## Data Model

```mermaid
erDiagram
    Campaign ||--o{ Promotion : contains
    Promotion ||--o{ Rule : "governed by"
    Promotion ||--o{ Redemption : "redeemed via"
    Promotion ||--o{ PromoCode : "accessed via"
    Customer ||--o{ Redemption : makes
    Segment ||--o{ Customer : contains
    Campaign }|--|| CampaignType : "classified as"
    
    Campaign {
        string campaignId PK
        string name
        string description
        date startDate
        date endDate
        string status
        string ownerId
        int budget
        string targetSegmentId
    }
    
    Promotion {
        string promotionId PK
        string campaignId FK
        string name
        string description
        string type
        date startDate
        date endDate
        string status
        json valueConfig
        int maxRedemptions
        int currentRedemptions
    }
    
    PromoCode {
        string codeId PK
        string promotionId FK
        string code
        boolean isUsed
        boolean isActive
        date expiryDate
    }
    
    Rule {
        string ruleId PK
        string promotionId FK
        string ruleType
        json ruleConfig
        int priority
    }
    
    Redemption {
        string redemptionId PK
        string promotionId FK
        string customerId FK
        string channelId
        timestamp redemptionTime
        json transactionDetails
        number discountAmount
        string status
    }
    
    Customer {
        string customerId PK
        string email
        string name
        json attributes
        array segmentIds
        date createdAt
        date lastActive
    }
    
    Segment {
        string segmentId PK
        string name
        string description
        json criteria
    }
    
    CampaignType {
        string typeId PK
        string name
        string description
    }
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Admin
    participant AdminAPI
    participant DB as DynamoDB
    participant PS as Promotion Service
    participant VS as Validation Service
    participant Queue as SQS
    participant RS as Redemption Service
    participant Customer
    participant AS as Analytics Service
    
    Admin->>AdminAPI: Create Promotion Campaign
    AdminAPI->>DB: Store Campaign Data
    AdminAPI->>PS: Generate Promotion Codes
    PS->>DB: Store Promotion Codes
    PS->>Queue: Publish Codes Created Event
    Queue->>AS: Process for Analytics
    
    Customer->>PS: Request Promotion
    PS->>VS: Validate Eligibility
    VS->>DB: Check Customer & Promo Status
    VS->>PS: Return Validation Result
    PS->>Customer: Return Promotion Details
    
    Customer->>RS: Redeem Promotion
    RS->>DB: Verify Code & Update Status
    RS->>DB: Record Redemption
    RS->>Queue: Publish Redemption Event
    Queue->>AS: Process for Analytics
    AS->>DB: Update Analytics Data
```
