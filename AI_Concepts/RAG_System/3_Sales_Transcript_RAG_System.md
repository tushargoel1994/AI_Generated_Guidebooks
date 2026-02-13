
## 6. Project: Sales Transcript RAG System

### Project Overview

Build a production-ready RAG system to enable sales teams to query historical sales call transcripts, extract insights, identify successful patterns, and improve sales techniques. The system will process the HuggingFace sales transcripts dataset and expose querying capabilities via REST API.

### Business Requirements

**Functional Requirements:**
- Ingest and process sales call transcripts
- Enable natural language querying of transcripts
- Identify successful sales patterns and objection handling
- Extract product-specific insights
- Generate sales coaching recommendations
- Support complex queries (multi-turn conversations, filtering by product/outcome)

**Non-Functional Requirements:**
- Response time: < 3 seconds for 95% of queries
- Support 100 concurrent users
- 99.9% uptime SLA
- GDPR compliant (customer data handling)
- Audit trail for all queries
- Cost per query: < $0.05

### Data Understanding

The sales transcripts dataset contains:
- Conversation transcripts between sales reps and customers
- Multiple turns per conversation
- Product categories discussed
- Sales outcomes (win/loss)
- Customer objections and responses
- Pricing discussions

**Key Attributes:**
- conversation_id
- transcript_text
- product_category
- outcome
- sales_rep_id (anonymized)
- timestamp
- duration
- objections_raised
- resolution_strategy

### Project File Architecture

```
sales-transcript-rag/
├── config/
│   ├── __init__.py
│   ├── settings.py
│   ├── aws_config.py
│   └── model_config.py
├── data/
│   ├── __init__.py
│   ├── loader.py
│   ├── preprocessor.py
│   ├── chunker.py
│   └── validator.py
├── embeddings/
│   ├── __init__.py
│   ├── embedding_service.py
│   ├── batch_processor.py
│   └── cache_manager.py
├── vectorstore/
│   ├── __init__.py
│   ├── opensearch_client.py
│   ├── index_manager.py
│   └── search_engine.py
├── retrieval/
│   ├── __init__.py
│   ├── query_processor.py
│   ├── retriever.py
│   ├── reranker.py
│   └── context_builder.py
├── generation/
│   ├── __init__.py
│   ├── llm_client.py
│   ├── prompt_templates.py
│   └── response_validator.py
├── guardrails/
│   ├── __init__.py
│   ├── input_validator.py
│   ├── output_filter.py
│   └── pii_detector.py
├── api/
│   ├── __init__.py
│   ├── app.py
│   ├── routes.py
│   ├── middleware.py
│   └── schemas.py
├── monitoring/
│   ├── __init__.py
│   ├── logger.py
│   ├── metrics.py
│   └── alerting.py
├── pipelines/
│   ├── __init__.py
│   ├── ingestion_pipeline.py
│   ├── query_pipeline.py
│   └── batch_update_pipeline.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── scripts/
│   ├── setup_infrastructure.py
│   ├── initial_ingestion.py
│   ├── create_indices.py
│   └── migrate_data.py
├── deployment/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── cloudformation/
│   └── terraform/
├── notebooks/
│   ├── exploration.ipynb
│   ├── evaluation.ipynb
│   └── performance_tuning.ipynb
├── requirements.txt
├── requirements-dev.txt
├── .env.example
├── .gitignore
├── README.md
└── Makefile
```

### Required Libraries

**Core Dependencies:**
```
boto3 (AWS SDK)
opensearch-py (OpenSearch client)
langchain (RAG orchestration framework)
transformers (embedding models)
sentence-transformers (semantic embeddings)
pandas (data manipulation)
numpy (numerical operations)
pydantic (data validation)
fastapi (API framework)
uvicorn (ASGI server)
python-dotenv (environment management)
```

**AWS-Specific:**
```
aws-lambda-powertools (Lambda utilities)
watchtower (CloudWatch logging)
botocore (AWS core functionality)
aioboto3 (async AWS operations)
```

**Vector Store & Search:**
```
opensearch-py-ml (ML plugin integration)
faiss-cpu (alternative vector search)
```

**NLP & ML:**
```
spacy (NLP processing)
nltk (text processing)
tiktoken (token counting)
```

**Testing & Quality:**
```
pytest (testing framework)
pytest-asyncio (async testing)
pytest-cov (coverage reporting)
hypothesis (property-based testing)
locust (load testing)
```

**Monitoring & Observability:**
```
prometheus-client (metrics)
opentelemetry-api (distributed tracing)
sentry-sdk (error tracking)
```

### Detailed File Descriptions

**config/settings.py**

Contains all application configuration including:
- Environment-specific settings (dev, staging, prod)
- Feature flags for experimental features
- Chunking parameters (size, overlap, strategy)
- Embedding model specifications
- Vector database connection details
- LLM provider and model selection
- Retrieval parameters (top_k, similarity threshold)
- API rate limits and quotas
- Logging levels and formats
- Cache TTL values

Implements configuration validation using Pydantic BaseSettings class. Supports environment variable overrides and secrets from AWS Secrets Manager.

**config/aws_config.py**

AWS service configuration including:
- Region and endpoint specifications
- S3 bucket names for different stages
- DynamoDB table names and capacity units
- OpenSearch cluster endpoints and authentication
- Bedrock model ARNs and configurations
- IAM role ARNs for cross-service access
- CloudWatch log group names
- SQS queue URLs
- EventBridge rule configurations

Includes helper functions for credential management and session handling with retry logic.

**config/model_config.py**

Model-specific configurations:
- Embedding model selection and parameters
- Dimension specifications
- Normalization strategies
- Batch sizes for embedding generation
- LLM model selection (Claude, Llama, etc.)
- Temperature and generation parameters
- Token limits for context and response
- Prompt template versions
- Model fallback chains

**data/loader.py**

Loads sales transcript data from HuggingFace:
- Downloads dataset using datasets library
- Implements streaming for large datasets
- Handles authentication if required
- Validates dataset schema
- Converts to pandas DataFrame for processing
- Implements caching to avoid repeated downloads
- Error handling for network issues
- Progress tracking for large downloads

Includes methods for incremental loading and filtering by date range or product category.

**data/preprocessor.py**

Cleans and prepares transcript data:
- Removes PII (phone numbers, email addresses, specific customer names)
- Normalizes text (lowercasing optional, whitespace cleanup)
- Handles special characters and encoding issues
- Splits multi-turn conversations into structured format
- Extracts speaker labels (sales rep vs customer)
- Identifies and tags objections and resolutions
- Enriches with derived metadata (sentiment, key phrases)
- Validates data quality and completeness
- Filters out low-quality or incomplete transcripts

Implements configurable preprocessing pipelines for different transcript formats.

**data/chunker.py**

Splits transcripts into retrievable chunks:
- Implements semantic chunking based on conversation turns
- Preserves multi-turn context windows
- Adds overlap to maintain continuity
- Respects token limits for embedding models
- Maintains parent-child relationships
- Includes metadata propagation to chunks
- Handles edge cases (very short or very long conversations)
- Generates chunk summaries for hierarchical retrieval
- Assigns unique chunk IDs with parent reference

Supports multiple chunking strategies selectable via configuration.

**data/validator.py**

Validates data quality throughout pipeline:
- Schema validation against expected structure
- Completeness checks (required fields present)
- Format validation (dates, IDs, enums)
- Range checks for numerical values
- Cross-field consistency validation
- Duplication detection
- Data type enforcement
- Custom business rule validation
- Generates quality score for each record
- Creates validation reports with issue details

Implements both synchronous and batch validation modes.

**embeddings/embedding_service.py**

Manages embedding generation:
- Abstracts embedding model providers (Bedrock, OpenAI, local)
- Implements connection pooling and session management
- Handles authentication and API key rotation
- Provides sync and async embedding methods
- Implements retry logic with exponential backoff
- Rate limiting to respect API quotas
- Token counting and truncation
- Batching for efficiency
- Error handling and logging
- Cost tracking per embedding call

Supports multiple embedding models with runtime switching.

**embeddings/batch_processor.py**

Processes large volumes of documents:
- Implements parallel processing with thread/process pools
- Batch size optimization based on model limits
- Progress tracking and checkpointing
- Resume capability for interrupted jobs
- Dead letter queue for failed items
- Implements backpressure handling
- Memory-efficient streaming processing
- Distributes work across multiple Lambda invocations
- Aggregates results and updates vector store
- Generates processing metrics and reports

**embeddings/cache_manager.py**

Caches embeddings to reduce costs:
- Content-based cache key generation (hash of text)
- MemoryDB/Redis integration for fast lookup
- S3 cold storage for long-term cache
- TTL management and cache invalidation
- Cache warming strategies
- Hit/miss ratio tracking
- Cache size monitoring and eviction policies
- Distributed cache consistency
- Bulk cache operations

**vectorstore/opensearch_client.py**

OpenSearch connection and operations:
- Client initialization with connection pooling
- Authentication handling (IAM, basic auth)
- Index creation and management
- Document indexing with bulk operations
- Vector search with k-NN queries
- Hybrid search combining vector and keyword
- Filtering by metadata fields
- Aggregation queries for analytics
- Error handling and retry logic
- Connection health monitoring

**vectorstore/index_manager.py**

Manages OpenSearch indices lifecycle:
- Index template definitions with mappings
- Creates indices with appropriate settings
- Configures k-NN algorithm (HNSW, IVF)
- Sets dimension and similarity function
- Implements index aliases for zero-downtime updates
- Reindexing strategies for schema changes
- Index optimization and force merge
- Monitoring index health and performance
- Implements hot-warm-cold tier management
- Backup and snapshot coordination

**vectorstore/search_engine.py**

High-level search interface:
- Abstracts search complexity from application
- Implements semantic search with embeddings
- Keyword search with BM25 scoring
- Hybrid search with score fusion (RRF or weighted)
- Faceted search for filtering
- Multi-field boosting strategies
- Query expansion and synonym handling
- Result pagination and sorting
- Explains search scores for transparency
- Implements query performance optimization

**retrieval/query_processor.py**

Processes user queries:
- Query understanding and intent detection
- Query expansion with synonyms
- Spell correction and fuzzy matching
- Extracts filters from natural language
- Generates structured search parameters
- Implements query rewriting for optimization
- Handles multi-part and complex queries
- Context tracking for conversational queries
- Query validation and sanitization
- Logs queries for analysis

**retrieval/retriever.py**

Core retrieval logic:
- Embeds user query using same model as documents
- Calls vector store search engine
- Implements multiple retrieval strategies
- Adjusts top_k based on query complexity
- Applies metadata filters
- Handles empty result scenarios
- Implements result diversification
- Deduplicates retrieved chunks
- Assembles retrieved context with metadata
- Tracks retrieval metrics

**retrieval/reranker.py**

Reranks retrieved results for relevance:
- Implements cross-encoder reranking
- Calculates relevance scores
- Diversity-aware reranking
- Business logic boosting (prioritize wins over losses)
- Recency boosting for time-sensitive queries
- Source authority scoring
- Applies final top_k selection
- Generates explainability features
- Performance-optimized for latency

**retrieval/context_builder.py**

Assembles final context for LLM:
- Formats retrieved chunks into prompt structure
- Implements token budget management
- Prioritizes chunks within token limit
- Adds source citations and metadata
- Creates structured context sections
- Handles hierarchical context (summaries + details)
- Implements context compression techniques
- Adds relevant system instructions
- Validates context quality before generation

**generation/llm_client.py**

Interfaces with LLM for generation:
- Bedrock client initialization and management
- Implements retry logic with exponential backoff
- Streams responses for long outputs
- Handles rate limiting and quotas
- Supports multiple models with fallback
- Implements cost tracking per request
- Token counting for input and output
- Timeout handling
- Error classification and logging
- A/B testing infrastructure

**generation/prompt_templates.py**

Manages prompt engineering:
- Template definitions for different query types
- Sales-specific prompts (pattern identification, coaching)
- System prompts with role definition
- Few-shot examples for consistency
- Dynamic prompt assembly based on context
- Version control for prompt iterations
- A/B testing different prompt variations
- Prompt optimization based on feedback
- Metadata injection into prompts
- Handles different LLM formatting requirements

**generation/response_validator.py**

Validates generated responses:
- Checks response grounding in retrieved context
- Detects potential hallucinations
- Validates citation accuracy
- Ensures response completeness
- Checks for toxicity or bias
- Validates JSON formatting if applicable
- Length validation
- Implements quality scoring
- Triggers regeneration if quality low
- Logs validation results

**guardrails/input_validator.py**

Validates user inputs:
- Detects prompt injection attempts
- Identifies jailbreak patterns
- Checks for malicious content
- Rate limiting per user
- Input length validation
- PII detection in queries
- Business logic validation (valid product names, etc.)
- Implements allow/deny lists
- Logs suspicious queries
- Returns user-friendly error messages

**guardrails/output_filter.py**

Filters generated outputs:
- Toxicity detection and filtering
- Bias detection in responses
- Ensures professional tone
- Validates no sensitive data leaked
- Checks compliance with business policies
- Implements content moderation
- Adds required disclaimers
- Filters competitor mentions if configured
- Logs filtered content for analysis

**guardrails/pii_detector.py**

Detects and handles PII:
- Uses regex patterns for common PII types
- Leverages Amazon Comprehend for detection
- Implements custom entity recognition
- Redaction strategies (masking, removal, tokenization)
- Context-aware PII handling
- Confidence scoring for detections
- Audit logging of PII exposure
- Configurable PII types and handling rules

**api/app.py**

FastAPI application setup:
- Application initialization and configuration
- Dependency injection setup
- CORS configuration
- Exception handlers (global and specific)
- Startup and shutdown events
- Health check endpoints
- Metrics exposure for Prometheus
- OpenAPI documentation configuration
- Authentication middleware integration
- Request ID generation and propagation

**api/routes.py**

API endpoint definitions:
- POST /query - Main RAG query endpoint
- GET /health - Health check
- GET /metrics - Prometheus metrics
- POST /feedback - User feedback collection
- GET /conversations/:id - Retrieve conversation history
- POST /batch-query - Batch query processing
- GET /sales-patterns - Aggregate insights
- POST /admin/reindex - Trigger reindexing
- Implements request validation with Pydantic
- Response models for consistency

**api/middleware.py**

Request processing middleware:
- Authentication and authorization
- Request logging with correlation IDs
- Rate limiting per API key
- Request/response timing
- Error handling and formatting
- CORS header management
- Request size validation
- Compression for large responses
- Security headers injection

**api/schemas.py**

Pydantic models for API:
- QueryRequest model with validation
- QueryResponse model with metadata
- Error response models
- Conversation history models
- Feedback submission models
- Admin operation models
- Nested models for complex structures
- Field validators for business logic
- Examples for documentation

**monitoring/logger.py**

Centralized logging:
- Structured JSON logging
- Integration with CloudWatch Logs
- Correlation ID tracking across services
- Log level configuration per module
- Context managers for request logging
- Performance logging (latency, throughput)
- Error logging with stack traces
- Sampling for high-volume logs
- Log aggregation and formatting

**monitoring/metrics.py**

Application metrics:
- Prometheus metric definitions
- Custom business metrics (queries per product, success rate)
- Latency histograms
- Counter metrics for events
- Gauge metrics for current state
- Metric labels for dimensionality
- Metric aggregation and rollups
- Integration with CloudWatch Metrics
- Alerting threshold definitions

**monitoring/alerting.py**

Alert management:
- Alert rule definitions
- SNS topic integration
- Alert severity classification
- Alert aggregation to reduce noise
- Alert routing based on severity
- On-call rotation integration
- Alert acknowledgment tracking
- Incident creation for critical alerts
- Alert testing and validation

**pipelines/ingestion_pipeline.py**

End-to-end ingestion orchestration:
- Coordinates loader, preprocessor, chunker
- Manages batch processing workflow
- Implements checkpointing for resumability
- Error handling with dead letter queue
- Progress tracking and reporting
- Parallel processing coordination
- Embedding generation integration
- Vector store indexing
- Metadata storage in DynamoDB
- Success/failure notifications

**pipelines/query_pipeline.py**

End-to-end query orchestration:
- Coordinates query processing through all stages
- Input validation via guardrails
- Query embedding generation
- Retrieval execution
- Reranking and context building
- LLM generation
- Output validation and filtering
- Response formatting
- Logging and metrics
- Error handling with fallbacks

**pipelines/batch_update_pipeline.py**

Handles bulk updates:
- Scheduled reindexing jobs
- Incremental data updates
- Schema migration support
- Index optimization runs
- Stale data cleanup
- Embedding model updates
- Performance tuning operations
- Bulk testing and validation
- Rollback capabilities

**tests/unit/**

Unit tests for individual components:
- Mocked external dependencies
- Fast execution for CI/CD
- High code coverage targets
- Tests for edge cases and error conditions
- Fixtures for test data
- Parameterized tests for multiple scenarios

**tests/integration/**

Integration tests:
- Tests interaction between components
- Uses test instances of AWS services
- Validates end-to-end data flow
- Tests error propagation
- Performance baseline tests
- Uses realistic test data

**tests/e2e/**

End-to-end system tests:
- Full API workflow tests
- User scenario simulation
- Load testing scripts
- Performance benchmarking
- Chaos engineering tests
- Production-like environment testing

**scripts/setup_infrastructure.py**

Infrastructure setup automation:
- Creates S3 buckets with proper configurations
- Sets up DynamoDB tables with indices
- Configures OpenSearch cluster
- Creates IAM roles and policies
- Establishes VPC and security groups
- Configures CloudWatch alarms
- Sets up SNS topics for alerts
- Idempotent operations for rerunning

**scripts/initial_ingestion.py**

First-time data load:
- Downloads dataset from HuggingFace
- Runs complete ingestion pipeline
- Validates all data processed
- Creates baseline indices
- Generates initial metrics
- Creates data quality report
- Estimated runtime and resource usage

**scripts/create_indices.py**

Index creation and management:
- Defines index schemas
- Creates indices with optimal settings
- Sets up aliases
- Configures index lifecycle policies
- Validates index creation
- Documents index purposes

**scripts/migrate_data.py**

Data migration utilities:
- Migrates between vector store versions
- Handles embedding model changes
- Schema evolution support
- Data transformation scripts
- Validation post-migration
- Rollback capabilities

**deployment/Dockerfile**

Container image definition:
- Multi-stage build for optimization
- Base image selection (Python 3.11 slim)
- Dependency installation with caching
- Application code copying
- Environment variable configuration
- Security hardening
- Non-root user execution
- Health check definition

**deployment/docker-compose.yml**

Local development setup:
- Application service definition
- Local OpenSearch container
- Redis cache container
- LocalStack for AWS services
- Volume mounts for code
- Network configuration
- Environment variables

**deployment/cloudformation/**

AWS CloudFormation templates:
- VPC and networking stack
- Compute resources (ECS, Lambda)
- Storage resources (S3, DynamoDB)
- OpenSearch cluster template
- IAM roles and policies
- CloudWatch resources
- Parameter store for configuration

**deployment/terraform/**

Infrastructure as Code:
- Modular Terraform structure
- Environment-specific variable files
- State management configuration
- Provider configurations
- Resource definitions matching CloudFormation

### Implementation Instructions

**Phase 1: Environment Setup (Week 1)**

1. Set up AWS account and configure IAM users/roles
2. Install Python 3.11+ and create virtual environment
3. Install all dependencies from requirements.txt
4. Configure AWS CLI with appropriate credentials
5. Set up development tools (IDE, linters, formatters)
6. Create .env file from .env.example with configurations
7. Run docker-compose up to start local services
8. Validate local environment with test scripts

**Phase 2: Infrastructure Deployment (Week 1-2)**

1. Review and customize CloudFormation templates
2. Deploy VPC and networking stack
3. Deploy S3 buckets for document storage
4. Deploy DynamoDB tables for metadata
5. Deploy OpenSearch cluster (start with dev-sized cluster)
6. Configure IAM roles for service-to-service access
7. Set up CloudWatch log groups and dashboards
8. Configure SNS topics for alerts
9. Test infrastructure connectivity and permissions
10. Document all resource ARNs and endpoints

**Phase 3: Data Ingestion Pipeline (Week 2-3)**

1. Implement data/loader.py to fetch HuggingFace dataset
2. Develop data/preprocessor.py with PII removal logic
3. Create data/chunker.py with semantic chunking
4. Build data/validator.py for quality checks
5. Implement embeddings/embedding_service.py with Bedrock
6. Develop embeddings/batch_processor.py for parallel processing
7. Create vectorstore/opensearch_client.py for connectivity
8. Implement vectorstore/index_manager.py to create indices
9. Build pipelines/ingestion_pipeline.py to orchestrate
10. Test with small dataset subset (100 transcripts)
11. Run full ingestion with monitoring
12. Validate all data indexed correctly

**Phase 4: Retrieval System (Week 3-4)**

1. Implement retrieval/query_processor.py for query handling
2. Create retrieval/retriever.py with semantic search
3. Develop vectorstore/search_engine.py with hybrid search
4. Implement retrieval/reranker.py with cross-encoder
5. Build retrieval/context_builder.py for prompt assembly
6. Test retrieval quality with sample queries
7. Tune retrieval parameters (top_k, similarity threshold)
8. Implement caching in embeddings/cache_manager.py
9. Optimize for latency and relevance
10. Create evaluation metrics for retrieval quality

**Phase 5: Generation System (Week 4-5)**

1. Implement generation/llm_client.py for Bedrock integration
2. Create generation/prompt_templates.py with sales prompts
3. Develop generation/response_validator.py for quality checks
4. Test generation with various query types
5. Implement streaming for long responses
6. Optimize token usage and cost
7. Create prompt variations for A/B testing
8. Validate response quality manually
9. Implement fallback mechanisms for failures
10. Build pipelines/query_pipeline.py to orchestrate

**Phase 6: Guardrails Implementation (Week 5)**

1. Implement guardrails/input_validator.py
2. Create guardrails/output_filter.py
3. Develop guardrails/pii_detector.py
4. Test with adversarial inputs
5. Tune thresholds to minimize false positives
6. Integrate into query pipeline
7. Monitor guardrail effectiveness
8. Document guardrail behaviors

**Phase 7: API Development (Week 5-6)**

1. Implement api/app.py with FastAPI
2. Create api/routes.py with all endpoints
3. Develop api/schemas.py with Pydantic models
4. Implement api/middleware.py for auth and logging
5. Create API documentation with examples
6. Test all endpoints with Postman/curl
7. Implement rate limiting
8. Add API key authentication
9. Test error handling scenarios
10. Deploy API to ECS or Lambda

**Phase 8: Monitoring & Observability (Week 6)**

1. Implement monitoring/logger.py with structured logging
2. Create monitoring/metrics.py with Prometheus metrics
3. Develop monitoring/alerting.py for critical issues
4. Set up CloudWatch dashboards
5. Configure alarms for SLA violations
6. Implement distributed tracing with X-Ray
7. Create runbooks for common issues
8. Test alerting with simulated failures
9. Document monitoring strategy

**Phase 9: Testing & Quality Assurance (Week 6-7)**

1. Write unit tests achieving 80%+ coverage
2. Create integration tests for all pipelines
3. Develop e2e tests for user workflows
4. Implement load testing with Locust
5. Conduct security testing (OWASP Top 10)
6. Perform penetration testing on API
7. Test disaster recovery procedures
8. Validate data retention policies
9. Test scaling under load
10. Create test report documenting results

**Phase 10: Production Deployment (Week 7-8)**

1. Review all configurations for production
2. Scale infrastructure appropriately
3. Deploy to production environment
4. Run smoke tests to validate
5. Enable monitoring and alerting
6. Conduct user acceptance testing
7. Train sales team on system usage
8. Create user documentation
9. Establish support procedures
10. Plan for ongoing maintenance

**Ongoing Operations:**

- Monitor system health daily
- Review cost reports weekly
- Analyze user feedback continuously
- Tune retrieval and generation based on metrics
- Update data weekly with new transcripts
- Conduct monthly performance reviews
- Plan quarterly feature enhancements
- Annual security audits
- Continuous model evaluation and updates

### Success Metrics

**Technical Metrics:**
- Query latency p95 < 3 seconds
- Retrieval precision @ 5 > 0.8
- Generation factuality score > 0.9
- System uptime > 99.9%
- Cost per query < $0.05

**Business Metrics:**
- User adoption rate > 70% of sales team
- Query volume growing 20% month-over-month
- User satisfaction score > 4.2/5
- Time saved per sales rep > 2 hours/week
- Sales improvement attribution > 5% increase

---
