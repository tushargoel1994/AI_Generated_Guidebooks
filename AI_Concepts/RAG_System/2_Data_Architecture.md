
## 5. Data Architecture for RAG Systems

### Document Lifecycle Management

**Ingestion Phase**

Documents flow through staged processing:

**Stage 1: Acquisition**
- S3 bucket receives documents via upload, API, or automated sync
- Supported formats: PDF, DOCX, TXT, HTML, JSON, CSV, MD
- Metadata extraction: filename, upload timestamp, source system, user/department
- Virus scanning and file validation
- DynamoDB entry created with status "pending"

**Stage 2: Preprocessing**
- Text extraction using format-specific parsers
- OCR for scanned documents via Amazon Textract
- Table extraction and preservation
- Image caption generation for embedded visuals
- Language detection and translation if needed
- Status updated to "processing"

**Stage 3: Enrichment**
- Named entity recognition (people, organizations, locations, dates)
- Topic classification and tagging
- Key phrase extraction
- Sentiment analysis for appropriate content
- Custom business metadata injection
- Quality scoring and filtering

**Stage 4: Chunking**
- Strategy selection based on document type
- Chunk size optimization (typically 512-1024 tokens)
- Overlap configuration (typically 10-20%)
- Metadata inheritance to chunks
- Parent-child relationship tracking

**Stage 5: Embedding**
- Batch embedding generation via Bedrock
- Dimension reduction if needed
- Normalization for cosine similarity
- Cache embeddings for reuse
- Store in vector database with metadata links

**Stage 6: Indexing**
- Vector index creation in OpenSearch
- Traditional indices for metadata in DynamoDB/RDS
- Graph relationships for document networks
- Search optimization and tuning
- Status updated to "indexed"

### Metadata Schema Design

**Document-Level Metadata**

```
document_id: unique identifier (UUID)
source_system: origin application or database
document_type: contract, email, report, transcript, etc.
title: human-readable name
author: creator or system
created_date: original creation timestamp
modified_date: last update timestamp
uploaded_date: ingestion timestamp
file_path: S3 location
file_size: bytes
file_format: extension
classification: public, internal, confidential
department: owning business unit
tags: searchable keywords
version: version number for updates
language: primary language code
access_control_list: user/group permissions
retention_policy: how long to keep
quality_score: automated quality assessment
processing_status: lifecycle stage
```

**Chunk-Level Metadata**

```
chunk_id: unique identifier (UUID)
document_id: parent document reference
chunk_index: position in document (0, 1, 2...)
chunk_text: actual content
chunk_tokens: token count
embedding_vector: dense representation
chunk_type: paragraph, table, list, heading
semantic_summary: AI-generated summary
entities: extracted named entities
topics: classified topics
page_number: source page if applicable
section: document section name
previous_chunk_id: sequential link
next_chunk_id: sequential link
embedding_model: model used for vector
embedding_date: when vector created
```

### Storage Architecture Patterns

**Hot-Warm-Cold Architecture**

**Hot Tier (OpenSearch + MemoryDB):**
- Recent documents (last 30 days)
- Frequently accessed content
- Low-latency retrieval required
- In-memory caching of popular queries
- Cost: High, Performance: Excellent

**Warm Tier (OpenSearch Standard Storage):**
- Medium-age documents (30-365 days)
- Occasionally accessed content
- Acceptable latency (100-500ms)
- Standard SSD storage
- Cost: Medium, Performance: Good

**Cold Tier (S3 + On-Demand Indexing):**
- Archive documents (1+ years)
- Rarely accessed content
- Higher latency acceptable (seconds)
- Re-index on access if needed
- Cost: Low, Performance: Adequate

**Data Partitioning Strategies**

**Time-Based Partitioning:**
Organize vectors by ingestion date for efficient archival and retrieval pattern optimization.

**Department/Tenant Partitioning:**
Separate indices per business unit for access control and performance isolation.

**Document Type Partitioning:**
Different indices for emails, contracts, reports allowing type-specific retrieval optimization.

**Hybrid Approach:**
Combine strategies with routing logic based on query context and user permissions.

### Update and Versioning Strategies

**Immutable Architecture (Recommended)**

When documents change:
- Create new version with new document_id
- Preserve old version with archived status
- Link versions via parent_version_id metadata
- Update vector database with new embedding
- Maintain version history in DynamoDB
- Set active_version flag for retrieval

**Benefits:** Complete audit trail, rollback capability, no vector corruption

**In-Place Update (Alternative)**

When documents change:
- Update existing document_id content
- Regenerate embeddings for changed chunks
- Update vector database entries
- Log change in audit table
- Risk of consistency issues during update

**Benefits:** Storage efficiency, simpler architecture

**Incremental Update Pattern**

For large documents:
- Track changes at chunk granularity
- Only re-embed modified sections
- Use content hashing to detect changes
- Maintain chunk-level version tracking
- Delta updates to vector database

### Data Quality and Validation

**Pre-Ingestion Validation**

- File integrity checks (checksums)
- Format compatibility verification
- Size and content limits enforcement
- Duplication detection via hashing
- Malware and security scanning

**Post-Ingestion Quality Metrics**

- Text extraction completeness percentage
- Embedding quality scores (distribution analysis)
- Metadata completeness percentage
- Entity extraction confidence
- Chunking boundary quality assessment

**Ongoing Data Health Monitoring**

- Orphaned vector detection (no metadata)
- Stale document identification
- Broken reference discovery
- Index fragmentation analysis
- Query performance degradation alerts

### Privacy and Compliance

**Data Residency**

- Region-specific S3 buckets and databases
- Cross-region replication for DR only
- Compliance with GDPR, CCPA requirements
- Data sovereignty controls

**Encryption Strategy**

- Encryption at rest: S3 SSE-KMS, EBS encryption, database encryption
- Encryption in transit: TLS 1.2+ for all connections
- Key rotation policies via AWS KMS
- Separate keys per classification level

**Right to Deletion**

- Comprehensive deletion workflow
- Vector database removal
- Metadata purging from all stores
- S3 object deletion with verification
- Audit log of deletion process
- Compliance certificate generation

**Access Logging**

- CloudTrail for all data operations
- Application-level access logs
- User query history tracking
- Document access patterns
- Anomaly detection on access patterns

---