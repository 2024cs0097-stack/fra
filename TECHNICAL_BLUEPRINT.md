# FRA ATLAS & WebGIS DSS - TECHNICAL BLUEPRINT
## AI-Powered Forest Rights Act Implementation Monitoring System

**Target States:** Madhya Pradesh, Odisha, Tripura, Telangana
**Version:** 1.0
**Date:** February 2026

---

## EXECUTIVE SUMMARY

This blueprint presents a cloud-native, microservices-based architecture for national-scale FRA monitoring. The system integrates existing OCR+NER modules, provisions for future AI asset mapping, and delivers a secure, scalable WebGIS platform for government decision-making.

**Key Differentiators:**
- Event-driven processing with queue-based orchestration
- Human-in-the-loop validation pipeline
- State-wise data isolation with national aggregation
- Pluggable AI module architecture
- Government-grade security and audit compliance

---

# SECTION 1: SYSTEM ARCHITECTURE

## 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        API GATEWAY (Kong/Nginx)                  │
│                    Authentication & Rate Limiting                │
└─────────────────────────────────────────────────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
┌───────▼────────┐    ┌─────────▼────────┐    ┌─────────▼─────────┐
│   Document     │    │   WebGIS         │    │   DSS Engine      │
│   Ingestion    │    │   Portal         │    │   Service         │
│   Service      │    │   (Frontend)     │    │                   │
└───────┬────────┘    └──────────────────┘    └─────────┬─────────┘
        │                                                 │
┌───────▼────────────────────────────────────────────────▼─────────┐
│              MESSAGE QUEUE (RabbitMQ / Redis Streams)            │
└──────────────────────────────────────────────────────────────────┘
        │                        │                        │
┌───────▼────────┐    ┌─────────▼────────┐    ┌─────────▼─────────┐
│  OCR+NER       │    │   GIS Processing │    │   Asset Mapping   │
│  Integration   │    │   Engine         │    │   Service         │
│  Service       │    │   (PostGIS)      │    │   (Future Plugin) │
└───────┬────────┘    └─────────┬────────┘    └─────────┬─────────┘
        │                        │                        │
        └────────────────────────┼────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│                    SUPABASE DATABASE LAYER                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ FRA Master   │  │ Spatial DB   │  │ Analytics    │          │
│  │ Database     │  │ (PostGIS)    │  │ Warehouse    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└──────────────────────────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│                    OBJECT STORAGE (Supabase Storage)            │
│              Documents | Shapefiles | Reports | Exports         │
└──────────────────────────────────────────────────────────────────┘
```

## 1.2 Microservices Breakdown

### Core Services

1. **API Gateway Service**
   - Technology: Kong Gateway / Nginx with Auth plugin
   - Responsibilities: Request routing, authentication, rate limiting, SSL termination
   - Endpoints: `/api/v1/*`

2. **Document Ingestion Service**
   - Technology: Node.js/Python FastAPI
   - Responsibilities: File upload, format validation, queue orchestration
   - Input: PDF/PNG/JPG (FRA documents)
   - Output: Job ID for tracking

3. **OCR+NER Integration Service** (Wrapper for existing module)
   - Technology: Python FastAPI
   - Responsibilities: Interface with existing OCR+NER module, normalize outputs
   - Input: Document binary + metadata
   - Output: Structured JSON

4. **Data Validation Service**
   - Technology: Python/Node.js with validation schemas
   - Responsibilities: Schema validation, entity normalization, geocoding
   - Output: Validated records with confidence scores

5. **GIS Processing Engine**
   - Technology: Python with GDAL, Shapely, PostGIS
   - Responsibilities: Polygon management, spatial indexing, overlap detection
   - Output: Spatial entities with topological validation

6. **DSS Engine Service**
   - Technology: Python with ML libraries (scikit-learn, pandas)
   - Responsibilities: Rule engine, scoring algorithms, scheme matching
   - Output: Recommendations, rankings, alerts

7. **Asset Mapping Service** (Future Plugin)
   - Technology: Python with TensorFlow/PyTorch
   - Responsibilities: Satellite imagery processing, feature extraction
   - API Contract: Defined, implementation pending

8. **Analytics Service**
   - Technology: Python with Pandas, aggregation logic
   - Responsibilities: Report generation, dashboard metrics
   - Output: JSON metrics, PDF/Excel reports

9. **Notification Service**
   - Technology: Node.js
   - Responsibilities: Email, SMS, in-app notifications
   - Triggers: Document processing completion, validation errors

10. **Audit Service**
    - Technology: Node.js with log aggregation
    - Responsibilities: All state changes, access logs, data lineage
    - Output: Immutable audit trail

## 1.3 Technology Stack

### Backend
- **Runtime**: Node.js (API services), Python (AI/GIS services)
- **Frameworks**: Express.js, FastAPI
- **Database**: Supabase (PostgreSQL + PostGIS)
- **Message Queue**: Redis Streams (lightweight) or RabbitMQ (production)
- **Storage**: Supabase Storage (S3-compatible)
- **Cache**: Redis

### Frontend
- **Framework**: React with TypeScript
- **Map Library**: MapLibre GL JS (open-source) or Leaflet
- **UI Components**: Shadcn/ui with Tailwind CSS
- **State Management**: Zustand/Redux Toolkit
- **Data Fetching**: React Query

### GIS Stack
- **Database**: PostGIS extension on Supabase
- **Processing**: GDAL, Shapely, GeoPandas
- **Tile Server**: Martin (PostGIS vector tiles) or pg_tileserv
- **Geocoding**: Custom gazetteer + OpenStreetMap Nominatim fallback

### AI/ML
- **Existing Module**: OCR+NER (provided as Python package/service)
- **Future ML**: PyTorch/TensorFlow for asset mapping
- **Serving**: FastAPI with model versioning

### DevOps
- **Containerization**: Docker
- **Orchestration**: Docker Compose (dev), Kubernetes (production)
- **CI/CD**: GitHub Actions
- **Monitoring**: Grafana + Prometheus
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana) or Loki

### Security
- **Authentication**: Supabase Auth with JWT
- **Authorization**: Row Level Security (RLS) + role-based policies
- **Encryption**: TLS 1.3, AES-256 at rest
- **Secrets**: Vault or Supabase Secrets

## 1.4 State-Wise Data Isolation

### Architectural Pattern: Multi-Tenant with Row-Level Security

```sql
-- Every table includes state_code for isolation
CREATE TABLE fra_claims (
    id UUID PRIMARY KEY,
    state_code VARCHAR(2) NOT NULL, -- MP, OD, TR, TG
    district_code VARCHAR(4) NOT NULL,
    -- other columns
);

-- RLS Policy Example
CREATE POLICY "State officials view own state"
ON fra_claims FOR SELECT
TO authenticated
USING (
    state_code = (auth.jwt() -> 'app_metadata' ->> 'state_code')
    OR
    (auth.jwt() -> 'app_metadata' ->> 'role') = 'national_admin'
);
```

### Data Routing
- API Gateway extracts user state from JWT
- All queries auto-filter by state_code
- National admins bypass filter
- Cross-state analytics aggregated in materialized views

## 1.5 Event-Driven Processing Flow

```
Document Upload
    ↓
[Queue: doc_uploaded]
    ↓
OCR+NER Processing → [Queue: extraction_complete]
    ↓
Schema Validation → [Queue: validation_complete]
    ↓
Geocoding → [Queue: geocoding_complete]
    ↓
Spatial Processing → [Queue: spatial_complete]
    ↓
Duplicate Detection → [Queue: dedup_complete]
    ↓
Conflict Detection → [Queue: conflict_check_complete]
    ↓
Database Insert → [Queue: record_created]
    ↓
Notification → User notified
```

### Queue Structure (Redis Streams)
```javascript
{
  "doc_uploaded": "stream:ingestion:uploaded",
  "extraction_complete": "stream:ocr:complete",
  "validation_complete": "stream:validation:complete",
  // Consumer groups for parallel processing
}
```

## 1.6 Deployment Architecture

### Cloud Provider: NIC (National Informatics Centre) Cloud or AWS GovCloud

```
┌─────────────────────────────────────────────────────────────────┐
│                         PRODUCTION ZONE                          │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │  Load Balancer │  │  API Gateway   │  │  Web Servers   │    │
│  │  (Multi-AZ)    │  │  Cluster       │  │  (Auto-scale)  │    │
│  └────────────────┘  └────────────────┘  └────────────────┘    │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │  Microservices │  │  Message Queue │  │  Worker Pools  │    │
│  │  Cluster       │  │  Cluster       │  │  (Spot/Scale)  │    │
│  └────────────────┘  └────────────────┘  └────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│                         DATABASE ZONE                            │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │  Supabase DB   │  │  Redis Cache   │  │  Object Store  │    │
│  │  (Multi-AZ)    │  │  Cluster       │  │  (Replicated)  │    │
│  └────────────────┘  └────────────────┘  └────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### Scalability Targets
- **Concurrent Users**: 10,000+
- **Documents/Day**: 50,000+
- **API Throughput**: 5,000 req/sec
- **Database Size**: 500GB-2TB (5 years)
- **Map Tile Requests**: 1M/day

---

# SECTION 2: DOCUMENT DIGITIZATION PIPELINE

## 2.1 Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 1: DOCUMENT INGESTION                                     │
└─────────────────────────────────────────────────────────────────┘
    User Upload (Web/Mobile/Bulk)
    ↓
    Format Check (PDF/PNG/JPG, max 25MB)
    ↓
    Virus Scan (ClamAV)
    ↓
    Store in Supabase Storage (raw_documents bucket)
    ↓
    Create Job Record (status: queued)
    ↓
    Emit Event: doc_uploaded

┌─────────────────────────────────────────────────────────────────┐
│ STAGE 2: OCR+NER PROCESSING (EXISTING MODULE)                   │
└─────────────────────────────────────────────────────────────────┘
    Worker consumes doc_uploaded event
    ↓
    Call Existing OCR+NER Module
    (Module Input: file_path, document_type)
    ↓
    Module processes and returns JSON
    ↓
    Store raw JSON (extraction_results bucket)
    ↓
    Update Job (status: extraction_complete)
    ↓
    Emit Event: extraction_complete

┌─────────────────────────────────────────────────────────────────┐
│ STAGE 3: SCHEMA VALIDATION                                      │
└─────────────────────────────────────────────────────────────────┘
    Worker consumes extraction_complete event
    ↓
    Validate against JSON Schema
    ↓
    Check Required Fields (claim_number, village_name, extent)
    ↓
    Flag Missing/Invalid Fields
    ↓
    Calculate Confidence Score (0-100)
    ↓
    Update Job (status: validated or needs_review)
    ↓
    Emit Event: validation_complete

┌─────────────────────────────────────────────────────────────────┐
│ STAGE 4: ENTITY NORMALIZATION                                   │
└─────────────────────────────────────────────────────────────────┘
    Normalize Names (Title Case, Trim)
    ↓
    Standardize Dates (ISO 8601)
    ↓
    Parse Coordinates (DMS → Decimal Degrees)
    ↓
    Normalize Land Extent (Hectares)
    ↓
    Map Claim Types (IFR/CFR/CR codes)
    ↓
    Emit Event: normalization_complete

┌─────────────────────────────────────────────────────────────────┐
│ STAGE 5: GEOCODING & SPATIAL VALIDATION                         │
└─────────────────────────────────────────────────────────────────┘
    Lookup Village in Gazetteer
    ↓
    Match State → District → Block → Village
    ↓
    Get Village Centroid / Boundary
    ↓
    If Coordinates Provided: Validate within village boundary
    ↓
    If No Coordinates: Use village centroid as placeholder
    ↓
    Create Point Geometry (ST_Point)
    ↓
    Emit Event: geocoding_complete

┌─────────────────────────────────────────────────────────────────┐
│ STAGE 6: DUPLICATE DETECTION                                    │
└─────────────────────────────────────────────────────────────────┘
    Check Existing Claims:
    - Same claim_number in same state
    - Same patta_holder + village (fuzzy match)
    - Overlapping coordinates (within 100m)
    ↓
    Calculate Duplicate Probability Score
    ↓
    Flag as potential_duplicate if score > 80%
    ↓
    Emit Event: dedup_complete

┌─────────────────────────────────────────────────────────────────┐
│ STAGE 7: CONFLICT DETECTION                                     │
└─────────────────────────────────────────────────────────────────┘
    Spatial Query:
    - Check intersection with Forest Boundaries
    - Check intersection with Revenue Land
    - Check intersection with Protected Areas
    - Check overlap with existing claims (>10% area)
    ↓
    Flag Conflicts with severity (low/medium/high)
    ↓
    Emit Event: conflict_check_complete

┌─────────────────────────────────────────────────────────────────┐
│ STAGE 8: HUMAN REVIEW (IF NEEDED)                               │
└─────────────────────────────────────────────────────────────────┘
    If confidence < 70% OR conflicts detected:
    ↓
    Route to Review Queue
    ↓
    District Officer reviews in Dashboard
    ↓
    Corrects fields, resolves conflicts
    ↓
    Marks as approved
    ↓
    Emit Event: review_approved

┌─────────────────────────────────────────────────────────────────┐
│ STAGE 9: DATABASE INSERTION                                     │
└─────────────────────────────────────────────────────────────────┘
    Insert into fra_claims table
    ↓
    Insert into patta_holders table
    ↓
    Create spatial polygon (if boundary data available)
    ↓
    Insert into claim_spatial_data table
    ↓
    Update village statistics
    ↓
    Emit Event: record_created

┌─────────────────────────────────────────────────────────────────┐
│ STAGE 10: POST-PROCESSING                                       │
└─────────────────────────────────────────────────────────────────┘
    Generate Record PDF
    ↓
    Update Analytics (state/district dashboards)
    ↓
    Send Notification to Uploader
    ↓
    Archive Original Document
    ↓
    Job Complete
```

## 2.2 OCR+NER Module Integration

### Expected JSON Schema from Existing Module

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["document_type", "extraction_timestamp", "entities"],
  "properties": {
    "document_type": {
      "type": "string",
      "enum": ["ifr_patta", "cfr_patta", "cr_patta", "claim_form"]
    },
    "extraction_timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "confidence_overall": {
      "type": "number",
      "minimum": 0,
      "maximum": 100
    },
    "entities": {
      "type": "object",
      "properties": {
        "claim_number": {
          "type": "object",
          "properties": {
            "value": {"type": "string"},
            "confidence": {"type": "number"},
            "bounding_box": {"type": "array"}
          }
        },
        "patta_holder": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "father_name": {"type": "string"},
            "confidence": {"type": "number"}
          }
        },
        "village": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "block": {"type": "string"},
            "district": {"type": "string"},
            "state": {"type": "string"},
            "confidence": {"type": "number"}
          }
        },
        "land_details": {
          "type": "object",
          "properties": {
            "extent_hectares": {"type": "number"},
            "survey_number": {"type": "string"},
            "coordinates": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "latitude": {"type": "number"},
                  "longitude": {"type": "number"}
                }
              }
            }
          }
        },
        "claim_status": {
          "type": "string",
          "enum": ["approved", "pending", "rejected", "partial"]
        },
        "approval_date": {
          "type": "string",
          "format": "date"
        },
        "administrative_details": {
          "type": "object",
          "properties": {
            "sdlc_name": {"type": "string"},
            "committee_decision_date": {"type": "string"},
            "district_committee_approval": {"type": "string"}
          }
        }
      }
    },
    "raw_text": {
      "type": "string",
      "description": "Full OCR text for fallback"
    },
    "processing_metadata": {
      "type": "object",
      "properties": {
        "model_version": {"type": "string"},
        "processing_time_ms": {"type": "number"},
        "page_count": {"type": "number"}
      }
    }
  }
}
```

### Integration API Design

**OCR+NER Integration Service Endpoints:**

```typescript
// POST /api/v1/ocr/process
{
  "job_id": "uuid",
  "document_url": "https://storage.../doc.pdf",
  "document_type": "ifr_patta",
  "metadata": {
    "state_code": "MP",
    "district_code": "MP01",
    "uploaded_by": "user_id"
  }
}

// Response
{
  "job_id": "uuid",
  "status": "processing",
  "estimated_completion": "2026-02-24T10:30:00Z"
}

// GET /api/v1/ocr/status/{job_id}
{
  "job_id": "uuid",
  "status": "completed",
  "result_url": "https://storage.../results.json",
  "confidence_overall": 92.5,
  "extraction_summary": {
    "claim_number": "MP/IFR/2024/12345",
    "patta_holder": "Ramesh Kumar",
    "village": "Bhopal Nagar"
  }
}
```

### Wrapper Service Implementation Strategy

```python
# ocr_integration_service.py

from fastapi import FastAPI, BackgroundTasks
from existing_ocr_module import OCRProcessor  # Existing module import

app = FastAPI()

class OCRIntegrationService:
    def __init__(self):
        self.ocr_processor = OCRProcessor()  # Initialize existing module

    async def process_document(self, job_id, document_path, doc_type):
        try:
            # Call existing module
            raw_result = self.ocr_processor.extract(
                file_path=document_path,
                document_type=doc_type
            )

            # Normalize to standard schema
            normalized_result = self.normalize_output(raw_result)

            # Store result
            await self.store_result(job_id, normalized_result)

            # Emit event
            await self.emit_event('extraction_complete', {
                'job_id': job_id,
                'result': normalized_result
            })

        except Exception as e:
            await self.handle_error(job_id, e)

    def normalize_output(self, raw_result):
        # Transform existing module output to standard schema
        # Handle any field mappings, type conversions
        return {
            "document_type": raw_result.get("type"),
            "entities": {
                "claim_number": {
                    "value": raw_result.get("claim_id"),
                    "confidence": raw_result.get("claim_id_confidence", 0)
                },
                # ... map all fields
            }
        }
```

## 2.3 Validation & Correction Dashboard

### Features

1. **Review Queue Interface**
   - List view: All documents needing review
   - Filters: State, district, confidence score, conflict type
   - Priority sorting: Low confidence first
   - Assignment: Assign to specific officers

2. **Document Review Screen**
   - Split view: Original document (left) | Extracted data (right)
   - Field-by-field editing with inline validation
   - Confidence score visualization (color-coded)
   - Bounding box overlay on document
   - Conflict alerts panel
   - Approve/Reject/Request Info actions

3. **Batch Operations**
   - Bulk approve high-confidence records
   - Export for offline review
   - Import corrections (CSV)

### UI Mockup Description

```
┌─────────────────────────────────────────────────────────────────┐
│ FRA Atlas | Document Review Dashboard                           │
├─────────────────────────────────────────────────────────────────┤
│ [Filters] State: MP  District: All  Status: Needs Review        │
│ [Search] Claim Number / Village                  [Sort: ▼]      │
├─────────────────────────────────────────────────────────────────┤
│ Documents Pending Review (142)                                  │
│                                                                  │
│ ┌───┬──────────────┬─────────────┬───────────┬────────────┐    │
│ │ # │ Claim Number │ Village     │ Confidence│ Issues     │    │
│ ├───┼──────────────┼─────────────┼───────────┼────────────┤    │
│ │ 1 │ MP/IFR/12345 │ Bhopal Nagar│ ██████ 68%│ Missing coords│ │
│ │ 2 │ MP/CFR/12346 │ Indore      │ ████ 45%  │ Duplicate? │    │
│ └───┴──────────────┴─────────────┴───────────┴────────────┘    │
│                                                                  │
│ [Click to review] →                                             │
└─────────────────────────────────────────────────────────────────┘

Review Screen:
┌──────────────────────────────────┬──────────────────────────────┐
│ Document Preview                 │ Extracted Data               │
│                                  │                              │
│ [PDF/Image Viewer]               │ Claim Number:                │
│                                  │ [MP/IFR/12345] ✓ 95%        │
│ [Zoom] [Rotate] [Download]       │                              │
│                                  │ Patta Holder:                │
│                                  │ [Ramesh Kumar] ⚠ 68%        │
│                                  │ [Edit] [Suggest Match]       │
│                                  │                              │
│                                  │ Village:                     │
│                                  │ [Bhopal Nagar] ✓ 92%        │
│                                  │ District: [Auto-filled]      │
│                                  │                              │
│                                  │ Land Extent:                 │
│                                  │ [2.5] hectares ✓ 88%        │
│                                  │                              │
│                                  │ Coordinates:                 │
│                                  │ ⚠ Not found in document     │
│                                  │ [Plot on Map] [Manual Entry] │
│                                  │                              │
│                                  │ ⚠ Alerts:                   │
│                                  │ • Overlaps with Claim #12340│
│                                  │ • Within Protected Area     │
│                                  │                              │
│                                  │ [Approve] [Reject] [Save]   │
└──────────────────────────────────┴──────────────────────────────┘
```

## 2.4 Data Flow Summary

```
Upload → Queue → OCR (Existing) → Validate → Geocode → Check Duplicates
→ Check Conflicts → [Human Review if needed] → Store → Notify
```

**SLAs:**
- Auto-processing: 2-5 minutes per document
- Human review queue: <24 hours
- Bulk upload: 1000 docs/hour

---

# SECTION 3: FRA ATLAS CREATION

## 3.1 Spatial Data Management

### Data Sources

1. **FRA Claim Boundaries**
   - Source: Survey data, GPS coordinates, hand-drawn maps
   - Format: Shapefiles, KML, GeoJSON, CSV with coordinates
   - Geometry: Point, Polygon, MultiPolygon

2. **Administrative Boundaries**
   - Source: Survey of India, state revenue departments
   - Levels: State, District, Block, Village
   - Format: Shapefiles

3. **Forest Boundaries**
   - Source: Forest Survey of India (FSI)
   - Types: Reserve Forest, Protected Forest, Unclassed Forest
   - Format: Shapefiles

4. **Other Layers**
   - Revenue land, Protected areas, Water bodies, Roads

### Shapefile Ingestion System

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│ SHAPEFILE UPLOAD INTERFACE                                      │
│ - Drag-drop .zip (SHP + SHX + DBF + PRJ)                       │
│ - Select layer type (FRA Boundary / Admin / Forest)            │
│ - Specify attributes mapping                                    │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ VALIDATION PIPELINE                                             │
│ 1. Check file completeness (.shp, .shx, .dbf, .prj)            │
│ 2. Validate CRS (must be EPSG:4326 or auto-convert)            │
│ 3. Check geometry validity (no self-intersections)             │
│ 4. Validate attribute schema                                    │
│ 5. Check for duplicate features                                 │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ TRANSFORMATION                                                  │
│ 1. Reproject to EPSG:4326 (WGS84)                              │
│ 2. Simplify geometry (Douglas-Peucker, tolerance=0.0001)       │
│ 3. Fix invalid geometries (ST_MakeValid)                       │
│ 4. Calculate area, centroid                                     │
│ 5. Generate tile pyramid for visualization                      │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ POSTGIS STORAGE                                                 │
│ - Insert into spatial tables                                    │
│ - Create spatial index (GIST)                                   │
│ - Update materialized views                                     │
│ - Generate vector tiles                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```python
# shapefile_processor.py

import geopandas as gpd
from shapely.geometry import mapping
from shapely.validation import make_valid

class ShapefileProcessor:
    def __init__(self, supabase_client):
        self.db = supabase_client

    def process_upload(self, zip_path, layer_type, attribute_mapping):
        # Read shapefile
        gdf = gpd.read_file(zip_path)

        # Validate CRS
        if gdf.crs != 'EPSG:4326':
            gdf = gdf.to_crs('EPSG:4326')

        # Validate geometries
        gdf['geometry'] = gdf['geometry'].apply(make_valid)

        # Simplify (reduce file size)
        gdf['geometry'] = gdf['geometry'].simplify(tolerance=0.0001)

        # Calculate attributes
        gdf['area_hectares'] = gdf.geometry.area * 111320 * 111320 / 10000
        gdf['centroid'] = gdf.geometry.centroid

        # Map attributes
        gdf = self.map_attributes(gdf, attribute_mapping)

        # Insert into PostGIS
        records = []
        for idx, row in gdf.iterrows():
            record = {
                'layer_type': layer_type,
                'geometry': mapping(row.geometry),
                'properties': row.drop('geometry').to_dict(),
                'area_hectares': row['area_hectares']
            }
            records.append(record)

        self.db.table('spatial_layers').insert(records).execute()

        # Trigger tile generation
        self.generate_tiles(layer_type)
```

## 3.2 GeoJSON Standardization

**Unified GeoJSON Schema:**

```json
{
  "type": "FeatureCollection",
  "name": "FRA Claims - Madhya Pradesh",
  "crs": {
    "type": "name",
    "properties": {"name": "EPSG:4326"}
  },
  "features": [
    {
      "type": "Feature",
      "id": "claim_uuid",
      "geometry": {
        "type": "Polygon",
        "coordinates": [[
          [77.123, 23.456],
          [77.124, 23.456],
          [77.124, 23.457],
          [77.123, 23.457],
          [77.123, 23.456]
        ]]
      },
      "properties": {
        "claim_number": "MP/IFR/2024/12345",
        "claim_type": "IFR",
        "patta_holder": "Ramesh Kumar",
        "village_name": "Bhopal Nagar",
        "village_code": "MP01001",
        "district": "Bhopal",
        "block": "Berasia",
        "state": "MP",
        "area_hectares": 2.5,
        "status": "approved",
        "approval_date": "2024-03-15",
        "centroid": [77.1235, 23.4565],
        "confidence_score": 92.5,
        "conflicts": [],
        "metadata": {
          "created_at": "2026-01-15T10:30:00Z",
          "updated_at": "2026-01-20T14:22:00Z",
          "source": "digitized",
          "verified": true
        }
      }
    }
  ]
}
```

## 3.3 Spatial Indexing with PostGIS

### Database Schema for Spatial Data

```sql
-- Spatial layers table
CREATE TABLE spatial_layers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    layer_type VARCHAR(50) NOT NULL, -- 'fra_ifr', 'fra_cfr', 'fra_cr', 'admin_district', 'forest_reserve'
    state_code VARCHAR(2) NOT NULL,
    name VARCHAR(255) NOT NULL,
    geometry GEOMETRY(Geometry, 4326) NOT NULL,
    properties JSONB,
    area_hectares DECIMAL(12, 4),
    centroid GEOMETRY(Point, 4326),
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    created_by UUID REFERENCES auth.users(id),
    version INTEGER DEFAULT 1
);

-- Spatial index
CREATE INDEX idx_spatial_layers_geom ON spatial_layers USING GIST(geometry);
CREATE INDEX idx_spatial_layers_centroid ON spatial_layers USING GIST(centroid);
CREATE INDEX idx_spatial_layers_type_state ON spatial_layers(layer_type, state_code);

-- FRA Claims with geometry
CREATE TABLE claim_spatial_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID REFERENCES fra_claims(id) ON DELETE CASCADE,
    geometry GEOMETRY(Polygon, 4326),
    boundary_source VARCHAR(50), -- 'gps_survey', 'digitized', 'estimated'
    survey_date DATE,
    surveyor_name VARCHAR(255),
    accuracy_meters DECIMAL(6, 2),
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_claim_spatial_geom ON claim_spatial_data USING GIST(geometry);

-- Boundary overlap detection (materialized view)
CREATE MATERIALIZED VIEW claim_overlaps AS
SELECT
    a.claim_id as claim_a,
    b.claim_id as claim_b,
    ST_Area(ST_Intersection(a.geometry, b.geometry)) * 111320 * 111320 / 10000 as overlap_hectares,
    (ST_Area(ST_Intersection(a.geometry, b.geometry)) / ST_Area(a.geometry)) * 100 as overlap_percentage
FROM claim_spatial_data a
JOIN claim_spatial_data b ON a.claim_id < b.claim_id
WHERE ST_Intersects(a.geometry, b.geometry)
AND ST_Area(ST_Intersection(a.geometry, b.geometry)) > 0;

CREATE INDEX idx_claim_overlaps_a ON claim_overlaps(claim_a);
CREATE INDEX idx_claim_overlaps_b ON claim_overlaps(claim_b);
```

## 3.4 Boundary Overlap Detection

### Spatial Queries

```sql
-- Find claims overlapping with protected areas
SELECT
    c.claim_number,
    c.patta_holder,
    pa.name as protected_area,
    ST_Area(ST_Intersection(cs.geometry, pa.geometry)) * 111320 * 111320 / 10000 as overlap_hectares
FROM claim_spatial_data cs
JOIN fra_claims c ON cs.claim_id = c.id
JOIN spatial_layers pa ON pa.layer_type = 'protected_area'
WHERE ST_Intersects(cs.geometry, pa.geometry)
AND cs.state_code = 'MP';

-- Find claims outside forest boundaries (potential issues)
SELECT
    c.claim_number,
    c.village_name,
    'Claim outside forest boundary' as issue
FROM claim_spatial_data cs
JOIN fra_claims c ON cs.claim_id = c.id
LEFT JOIN spatial_layers fb ON fb.layer_type = 'forest_boundary'
    AND ST_Within(cs.geometry, fb.geometry)
WHERE fb.id IS NULL;
```

## 3.5 Dispute Tagging System

### Dispute Types

1. **Boundary Disputes**
   - Overlapping claims
   - Claims outside designated forest area
   - Encroachment on protected areas

2. **Documentation Disputes**
   - Missing evidence
   - Conflicting records
   - Invalid survey data

3. **Administrative Disputes**
   - Pending committee approval
   - Cross-jurisdictional claims
   - Title conflicts

### Implementation

```sql
CREATE TABLE disputes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dispute_type VARCHAR(50) NOT NULL,
    severity VARCHAR(20) NOT NULL, -- 'low', 'medium', 'high', 'critical'
    claim_id UUID REFERENCES fra_claims(id),
    conflicting_claim_id UUID REFERENCES fra_claims(id),
    description TEXT,
    geometry GEOMETRY(Geometry, 4326), -- disputed boundary
    status VARCHAR(50) DEFAULT 'open', -- 'open', 'under_review', 'resolved', 'escalated'
    assigned_to UUID REFERENCES auth.users(id),
    resolution_notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    resolved_at TIMESTAMPTZ
);

CREATE INDEX idx_disputes_claim ON disputes(claim_id);
CREATE INDEX idx_disputes_status ON disputes(status);
```

## 3.6 Version Control for Land Records

### Versioning Strategy

```sql
-- Audit trail for spatial changes
CREATE TABLE spatial_layer_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    layer_id UUID REFERENCES spatial_layers(id),
    version INTEGER NOT NULL,
    geometry GEOMETRY(Geometry, 4326),
    properties JSONB,
    changed_by UUID REFERENCES auth.users(id),
    change_type VARCHAR(50), -- 'created', 'boundary_updated', 'attributes_updated', 'deleted'
    change_reason TEXT,
    changed_at TIMESTAMPTZ DEFAULT now()
);

-- Function to auto-version on update
CREATE OR REPLACE FUNCTION version_spatial_layer()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO spatial_layer_history (
        layer_id, version, geometry, properties, changed_by, change_type
    ) VALUES (
        OLD.id, OLD.version, OLD.geometry, OLD.properties,
        NEW.updated_by, 'boundary_updated'
    );
    NEW.version = OLD.version + 1;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER spatial_layer_version_trigger
BEFORE UPDATE ON spatial_layers
FOR EACH ROW
EXECUTE FUNCTION version_spatial_layer();
```

---

# SECTION 4: FUTURE ASSET MAPPING INTEGRATION

## 4.1 Pluggable Architecture Design

### Asset Mapping Service API Contract

**Service Specification:**

```yaml
service: asset-mapping-service
version: 1.0.0
description: AI-powered satellite imagery analysis for asset detection

endpoints:
  - path: /api/v1/assets/analyze
    method: POST
    description: Analyze satellite imagery for asset detection
    input:
      tile_reference: string (e.g., "MP_Bhopal_2024_Q1")
      bbox: [min_lon, min_lat, max_lon, max_lat]
      asset_types: array<string> (e.g., ["water", "agriculture", "forest"])
      imagery_source: string (e.g., "sentinel2", "landsat8")
      imagery_date: date
    output:
      job_id: uuid
      status: string
      estimated_completion: datetime

  - path: /api/v1/assets/results/{job_id}
    method: GET
    description: Retrieve asset detection results
    output:
      job_id: uuid
      status: string (pending/processing/completed/failed)
      result_url: string (GeoJSON download link)
      layers: array<LayerMetadata>
      processing_metadata:
        model_version: string
        confidence_threshold: number
        processing_time_seconds: number

  - path: /api/v1/assets/layers/register
    method: POST
    description: Register detected assets as map layer
    input:
      job_id: uuid
      layer_name: string
      visibility: string (public/private/state-restricted)
      auto_refresh: boolean
```

### GeoJSON Output Schema for Asset Layers

```json
{
  "type": "FeatureCollection",
  "metadata": {
    "layer_type": "water_bodies",
    "analysis_date": "2024-03-15",
    "imagery_source": "Sentinel-2",
    "model_version": "water-detect-v2.1",
    "confidence_threshold": 0.85,
    "bbox": [77.0, 23.0, 78.0, 24.0]
  },
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Polygon",
        "coordinates": [[...]]
      },
      "properties": {
        "asset_type": "water_body",
        "asset_subtype": "pond",
        "area_sqm": 15000,
        "confidence_score": 0.92,
        "detected_at": "2024-03-15T10:30:00Z",
        "change_detected": true,
        "previous_area_sqm": 12000,
        "village_code": "MP01001",
        "accessibility": "accessible",
        "seasonal": false
      }
    }
  ]
}
```

## 4.2 Integration Points

### Database Schema for Asset Layers

```sql
-- Asset detection jobs
CREATE TABLE asset_mapping_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    state_code VARCHAR(2) NOT NULL,
    bbox GEOMETRY(Polygon, 4326),
    asset_types VARCHAR[] NOT NULL,
    imagery_source VARCHAR(50),
    imagery_date DATE,
    status VARCHAR(50) DEFAULT 'queued',
    result_url TEXT,
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    completed_at TIMESTAMPTZ
);

-- Detected assets
CREATE TABLE detected_assets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID REFERENCES asset_mapping_jobs(id),
    asset_type VARCHAR(50) NOT NULL,
    geometry GEOMETRY(Geometry, 4326) NOT NULL,
    properties JSONB,
    confidence_score DECIMAL(5, 4),
    village_code VARCHAR(10),
    detected_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_detected_assets_geom ON detected_assets USING GIST(geometry);
CREATE INDEX idx_detected_assets_type ON detected_assets(asset_type);

-- Change detection log
CREATE TABLE asset_change_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id UUID REFERENCES detected_assets(id),
    change_type VARCHAR(50), -- 'area_increase', 'area_decrease', 'new_asset', 'removed'
    previous_geometry GEOMETRY(Geometry, 4326),
    new_geometry GEOMETRY(Geometry, 4326),
    change_magnitude DECIMAL(10, 2),
    detected_at TIMESTAMPTZ DEFAULT now()
);
```

## 4.3 Layer Registration System

### Auto-Layer Refresh

```javascript
// asset_layer_manager.js

class AssetLayerManager {
  async registerLayer(jobId, layerConfig) {
    const job = await db.table('asset_mapping_jobs')
      .select('*')
      .eq('id', jobId)
      .single();

    // Download GeoJSON from result_url
    const geoJSON = await fetch(job.result_url).then(r => r.json());

    // Insert features into detected_assets
    const features = geoJSON.features.map(f => ({
      job_id: jobId,
      asset_type: f.properties.asset_type,
      geometry: f.geometry,
      properties: f.properties,
      confidence_score: f.properties.confidence_score
    }));

    await db.table('detected_assets').insert(features);

    // Create map layer entry
    const layerId = await db.table('map_layers').insert({
      name: layerConfig.layer_name,
      type: 'asset_detection',
      source_type: 'asset_mapping',
      source_job_id: jobId,
      visibility: layerConfig.visibility,
      auto_refresh: layerConfig.auto_refresh,
      refresh_interval_days: 30,
      style: this.generateStyle(geoJSON.metadata.layer_type)
    });

    return layerId;
  }

  async refreshLayer(layerId) {
    const layer = await db.table('map_layers')
      .select('*')
      .eq('id', layerId)
      .single();

    // Trigger new asset mapping job for same bbox
    const originalJob = await db.table('asset_mapping_jobs')
      .select('*')
      .eq('id', layer.source_job_id)
      .single();

    const newJob = await this.triggerAssetMapping({
      bbox: originalJob.bbox,
      asset_types: originalJob.asset_types,
      imagery_source: originalJob.imagery_source
    });

    // Detect changes
    await this.detectChanges(layer.source_job_id, newJob.id);
  }

  async detectChanges(oldJobId, newJobId) {
    // Compare geometries between old and new detections
    // Log significant changes
    const query = `
      INSERT INTO asset_change_log (asset_id, change_type, previous_geometry, new_geometry, change_magnitude)
      SELECT
        new.id,
        CASE
          WHEN old.id IS NULL THEN 'new_asset'
          WHEN ST_Area(new.geometry) > ST_Area(old.geometry) * 1.2 THEN 'area_increase'
          WHEN ST_Area(new.geometry) < ST_Area(old.geometry) * 0.8 THEN 'area_decrease'
          ELSE 'no_change'
        END as change_type,
        old.geometry as previous_geometry,
        new.geometry as new_geometry,
        ABS(ST_Area(new.geometry) - ST_Area(old.geometry)) as change_magnitude
      FROM detected_assets new
      LEFT JOIN detected_assets old ON ST_Intersects(new.geometry, old.geometry)
        AND old.job_id = $1
      WHERE new.job_id = $2
      AND (old.id IS NULL OR ABS(ST_Area(new.geometry) - ST_Area(old.geometry)) > 100);
    `;

    await db.raw(query, [oldJobId, newJobId]);
  }
}
```

## 4.4 Placeholder Service Structure

```
/services/asset-mapping-service/
├── README.md
├── Dockerfile
├── requirements.txt
├── src/
│   ├── api/
│   │   ├── __init__.py
│   │   ├── main.py              # FastAPI app
│   │   ├── routes/
│   │   │   ├── analyze.py       # POST /analyze endpoint
│   │   │   ├── results.py       # GET /results endpoint
│   │   │   └── layers.py        # POST /layers/register
│   │   └── models/
│   │       ├── requests.py      # Pydantic request models
│   │       └── responses.py     # Pydantic response models
│   ├── core/
│   │   ├── config.py
│   │   ├── ml_pipeline.py       # ML inference pipeline (placeholder)
│   │   └── geojson_builder.py
│   └── utils/
│       ├── imagery_fetcher.py   # Satellite data download
│       └── storage.py
├── tests/
│   └── test_api.py
└── docker-compose.yml
```

### Placeholder Implementation

```python
# src/api/routes/analyze.py

from fastapi import APIRouter, HTTPException
from ..models.requests import AnalyzeRequest
from ..models.responses import AnalyzeResponse

router = APIRouter()

@router.post("/analyze", response_model=AnalyzeResponse)
async def analyze_imagery(request: AnalyzeRequest):
    """
    PLACEHOLDER: Future AI asset mapping implementation

    This endpoint will:
    1. Fetch satellite imagery for the given bbox and date
    2. Run ML model inference for asset detection
    3. Generate GeoJSON output
    4. Store results and return job_id
    """

    # For now, return mock response
    return AnalyzeResponse(
        job_id="mock-job-id",
        status="pending",
        estimated_completion="2026-02-24T12:00:00Z",
        message="Asset mapping service not yet implemented. Placeholder response."
    )

# When implemented, this will call:
# - imagery_fetcher.download_sentinel2(bbox, date)
# - ml_pipeline.run_inference(imagery, asset_types)
# - geojson_builder.create_output(detections)
```

---

# SECTION 5: WEBGIS PORTAL DESIGN

## 5.1 Portal Architecture

### Technology Stack

- **Frontend Framework**: React 18 with TypeScript
- **Map Library**: MapLibre GL JS (open-source, vector tiles)
- **UI Components**: Shadcn/ui + Tailwind CSS
- **State Management**: Zustand for global state
- **Data Fetching**: React Query (TanStack Query)
- **Charts**: Recharts / Chart.js
- **Tables**: TanStack Table
- **Forms**: React Hook Form + Zod validation
- **i18n**: react-i18next (multilingual)

### Application Structure

```
/src/
├── app/
│   ├── layout.tsx              # Root layout with auth
│   ├── page.tsx                # Landing page
│   ├── dashboard/              # Main dashboard
│   │   ├── page.tsx
│   │   ├── layout.tsx
│   │   └── [role]/             # Role-based dashboards
│   │       ├── national/
│   │       ├── state/
│   │       ├── district/
│   │       └── block/
│   ├── atlas/                  # FRA Atlas viewer
│   │   ├── page.tsx
│   │   └── components/
│   ├── village/                # Village profile
│   │   └── [id]/
│   ├── claims/                 # Claims management
│   ├── disputes/               # Dispute resolution
│   ├── reports/                # Report generation
│   └── admin/                  # System administration
├── components/
│   ├── ui/                     # Shadcn components
│   ├── map/                    # Map components
│   │   ├── MapViewer.tsx
│   │   ├── LayerControl.tsx
│   │   ├── TimelineSlider.tsx
│   │   └── DrawingTools.tsx
│   ├── charts/                 # Chart components
│   ├── tables/                 # Table components
│   └── layout/                 # Layout components
├── lib/
│   ├── supabase.ts             # Supabase client
│   ├── map-utils.ts            # Map utilities
│   └── api.ts                  # API client
├── hooks/                      # Custom hooks
├── stores/                     # Zustand stores
├── types/                      # TypeScript types
└── utils/                      # Utility functions
```

## 5.2 Role-Based Dashboards

### Dashboard Matrix

| Role | View Access | Edit Rights | Analytics Scope | Key Features |
|------|-------------|-------------|-----------------|--------------|
| **National Admin (MoTA)** | All states | System config | National | Cross-state comparison, policy simulation |
| **State Admin** | Own state | State data | State-wide | State progress, district ranking |
| **District Collector** | Own district | Approve claims | District | Claim approval queue, village dashboard |
| **Block Officer** | Own block | Data entry | Block | Document upload, field verification |
| **Forest Officer** | Overlapping areas | Forest data | Jurisdiction | Conflict resolution, boundary validation |
| **Tribal Welfare Officer** | Beneficiary data | CSS schemes | State/District | Scheme enrollment, beneficiary tracking |

### Dashboard Layouts

#### 1. National Admin Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│ FRA Atlas | National Dashboard                     [User Menu]   │
├─────────────────────────────────────────────────────────────────┤
│ [National Overview] [State Comparison] [Policy Tools] [Reports] │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Key Metrics (All 4 States)                                      │
│ ┌──────────────┬──────────────┬──────────────┬──────────────┐  │
│ │ Total Claims │ Approved     │ Area Titled  │ Disputes     │  │
│ │ 1,245,892    │ 89.2%        │ 4.2M ha      │ 1,234 open   │  │
│ └──────────────┴──────────────┴──────────────┴──────────────┘  │
│                                                                  │
│ ┌────────────────────────────┬───────────────────────────────┐ │
│ │ State Performance Map      │ Progress Timeline             │ │
│ │                            │                               │ │
│ │ [Interactive India Map]    │ [Line chart: Claims over time]│ │
│ │ • MP: 92% complete         │                               │ │
│ │ • OD: 87% complete         │ By State:                     │ │
│ │ • TR: 94% complete         │ ─── MP                        │ │
│ │ • TG: 85% complete         │ ─── OD                        │ │
│ │                            │ ─── TR                        │ │
│ │                            │ ─── TG                        │ │
│ └────────────────────────────┴───────────────────────────────┘ │
│                                                                  │
│ State-wise Breakdown                                            │
│ ┌──────┬────────┬─────────┬─────────┬──────────┬─────────┐    │
│ │ State│ Claims │ Approved│ Rejected│ Area (ha)│ Progress│    │
│ ├──────┼────────┼─────────┼─────────┼──────────┼─────────┤    │
│ │ MP   │ 450K   │ 92%     │ 3%      │ 1.8M     │████████░│    │
│ │ OD   │ 380K   │ 87%     │ 5%      │ 1.5M     │███████░░│    │
│ │ TR   │ 215K   │ 94%     │ 2%      │ 0.5M     │█████████│    │
│ │ TG   │ 200K   │ 85%     │ 6%      │ 0.4M     │███████░░│    │
│ └──────┴────────┴─────────┴─────────┴──────────┴─────────┘    │
│                                                                  │
│ [View Detailed Reports] [Export National Data] [Policy Sim]    │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. State Admin Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│ FRA Atlas | Madhya Pradesh Dashboard          [User: State Admin]│
├─────────────────────────────────────────────────────────────────┤
│ [Overview] [Districts] [FRA Atlas] [Disputes] [CSS Integration] │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Madhya Pradesh - Progress Summary                              │
│ ┌──────────────┬──────────────┬──────────────┬──────────────┐  │
│ │ Total Claims │ This Month   │ Pending      │ Disputes     │  │
│ │ 450,234      │ +2,145       │ 8,234        │ 145 open     │  │
│ │              │ ↑ 12%        │              │ 23 new       │  │
│ └──────────────┴──────────────┴──────────────┴──────────────┘  │
│                                                                  │
│ ┌────────────────────────────┬───────────────────────────────┐ │
│ │ District Map               │ Claim Type Distribution       │ │
│ │                            │                               │ │
│ │ [Choropleth Map of MP]     │ [Pie Chart]                   │ │
│ │ Color: Approval %          │ • IFR: 65%                    │ │
│ │                            │ • CFR: 28%                    │ │
│ │ Hover: District stats      │ • CR: 7%                      │ │
│ │                            │                               │ │
│ └────────────────────────────┴───────────────────────────────┘ │
│                                                                  │
│ District Performance Ranking                                    │
│ ┌──────────────┬────────┬─────────┬─────────┬──────────────┐  │
│ │ District     │ Claims │ Approved│ Pending │ Action       │  │
│ ├──────────────┼────────┼─────────┼─────────┼──────────────┤  │
│ │ Bhopal       │ 45K    │ 95% ✓   │ 2,100   │ [View]       │  │
│ │ Indore       │ 52K    │ 93% ✓   │ 3,400   │ [View]       │  │
│ │ Jabalpur     │ 38K    │ 88% ⚠   │ 4,200   │ [Review]     │  │
│ │ ...          │        │         │         │              │  │
│ └──────────────┴────────┴─────────┴─────────┴──────────────┘  │
│                                                                  │
│ Recent Alerts                                                   │
│ • 23 new boundary disputes in Dindori district                 │
│ • 145 claims pending approval >90 days in Mandla               │
│ • Asset mapping completed for Balaghat (view results)          │
│                                                                  │
│ [Generate State Report] [View Full Atlas] [Manage Users]       │
└─────────────────────────────────────────────────────────────────┘
```

#### 3. District Collector Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│ FRA Atlas | Bhopal District                [User: DC Bhopal]   │
├─────────────────────────────────────────────────────────────────┤
│ [Claim Queue] [Village View] [Map] [Disputes] [Reports]        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Approval Queue (Requires Your Action)                 [142]    │
│ ┌──────────────────────────────────────────────────────────────┐│
│ │ [Filter: Block ▼] [Status ▼] [Priority ▼]    [Search...]    ││
│ ├──────────────────────────────────────────────────────────────┤│
│ │ High Priority (45)                                           ││
│ │ ┌────────────────────────────────────────────────────────┐  ││
│ │ │ MP/IFR/2024/12345 | Ramesh Kumar | Berasia Block      │  ││
│ │ │ Village: Bhopal Nagar | Area: 2.5 ha | Submitted: 92d │  ││
│ │ │ ⚠ Overlaps with protected area                        │  ││
│ │ │ [Review] [Approve] [Reject] [Request Info]            │  ││
│ │ └────────────────────────────────────────────────────────┘  ││
│ │ ┌────────────────────────────────────────────────────────┐  ││
│ │ │ MP/CFR/2024/12346 | Village: Indore | Area: 15 ha     │  ││
│ │ │ 23 families | Submitted: 85d                          │  ││
│ │ │ ✓ All documents verified                              │  ││
│ │ │ [Review] [Approve] [Reject]                           │  ││
│ │ └────────────────────────────────────────────────────────┘  ││
│ │                                                              ││
│ │ [Load More] [Bulk Approve Selected (0)]                     ││
│ └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│ ┌────────────────────────────┬───────────────────────────────┐ │
│ │ District Map View          │ Block Statistics              │ │
│ │                            │                               │ │
│ │ [Interactive Map]          │ ┌─────────┬──────┬─────────┐ │ │
│ │ Toggle Layers:             │ │ Block   │Claims│ Pending │ │ │
│ │ ☑ FRA Claims               │ ├─────────┼──────┼─────────┤ │ │
│ │ ☑ Villages                 │ │ Berasia │ 4.5K │ 124     │ │ │
│ │ ☑ Forest Boundaries        │ │ Huzur   │ 3.2K │ 89      │ │ │
│ │ ☐ Water Bodies             │ │ Phanda  │ 2.8K │ 156     │ │ │
│ │ ☐ Revenue Boundaries       │ └─────────┴──────┴─────────┘ │ │
│ │                            │                               │ │
│ │ [Draw Tool] [Measure]      │ [View All Blocks]             │ │
│ └────────────────────────────┴───────────────────────────────┘ │
│                                                                  │
│ [Export Pending Claims] [Generate District Report]             │
└─────────────────────────────────────────────────────────────────┘
```

#### 4. Block Officer Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│ FRA Atlas | Berasia Block                  [User: BO Berasia]  │
├─────────────────────────────────────────────────────────────────┤
│ [Upload Documents] [My Submissions] [Village List] [Field App] │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Document Upload Center                                         │
│ ┌──────────────────────────────────────────────────────────────┐│
│ │                                                              ││
│ │   [Drag & drop files here or click to browse]               ││
│ │                                                              ││
│ │   Supported: PDF, PNG, JPG (max 25MB each)                  ││
│ │                                                              ││
│ │   Bulk upload up to 50 documents at once                    ││
│ │                                                              ││
│ └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│ Document Type: [IFR Patta ▼]  Village: [Select... ▼]          │
│ Notes: [Optional notes about this batch]                       │
│                                                                  │
│ [Upload] [Cancel]                                              │
│                                                                  │
│ Recent Submissions (Last 7 days)                               │
│ ┌────────────┬───────────────┬──────────┬─────────────────┐   │
│ │ Date       │ Document      │ Status   │ Action          │   │
│ ├────────────┼───────────────┼──────────┼─────────────────┤   │
│ │ 2026-02-23 │ IFR_batch_45  │ ✓ Done   │ [View Records]  │   │
│ │ 2026-02-23 │ CFR_village12 │ ⏳ Process│ [Track]        │   │
│ │ 2026-02-22 │ IFR_batch_44  │ ⚠ Review │ [Fix Issues]    │   │
│ └────────────┴───────────────┴──────────┴─────────────────┘   │
│                                                                  │
│ My Block Statistics                                            │
│ ┌──────────────┬──────────────┬──────────────┐                │
│ │ Uploaded     │ Processed    │ Approved     │                │
│ │ 234 (this mo)│ 218          │ 195          │                │
│ └──────────────┴──────────────┴──────────────┘                │
│                                                                  │
│ [Download Mobile App] [Training Resources]                     │
└─────────────────────────────────────────────────────────────────┘
```

#### 5. Forest Officer Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│ FRA Atlas | Forest Division - Bhopal      [User: DFO Bhopal]   │
├─────────────────────────────────────────────────────────────────┤
│ [Forest Map] [Boundary Conflicts] [Joint Verification] [Reports]│
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Boundary Conflict Resolution                          [45 Open]│
│ ┌──────────────────────────────────────────────────────────────┐│
│ │ [Filter: Severity ▼] [Forest Type ▼]        [Search...]     ││
│ │                                                              ││
│ │ Critical (12)                                                ││
│ │ ┌────────────────────────────────────────────────────────┐  ││
│ │ │ Conflict #CF-2024-145                                  │  ││
│ │ │ Location: Bhopal Nagar Village                         │  ││
│ │ │ Issue: CFR claim overlaps Reserved Forest (12.5 ha)    │  ││
│ │ │ Claim: MP/CFR/2024/12346 (23 families)                │  ││
│ │ │                                                        │  ││
│ │ │ [View on Map] [Joint Verification] [Recommend]        │  ││
│ │ └────────────────────────────────────────────────────────┘  ││
│ │                                                              ││
│ └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│ ┌────────────────────────────┬───────────────────────────────┐ │
│ │ Forest Overlay Map         │ Verification Schedule         │ │
│ │                            │                               │ │
│ │ [Map showing:]             │ Upcoming Joint Verifications: │ │
│ │ • Forest boundaries (green)│                               │ │
│ │ • FRA claims (blue)        │ • 25 Feb: Bhopal Nagar (CFR)  │ │
│ │ • Conflicts (red outline)  │ • 27 Feb: Indore Village      │ │
│ │                            │ • 2 Mar: Survey pending (5)   │ │
│ │ [Measure] [Draw]           │                               │ │
│ │ [Export Shapefiles]        │ [Schedule New] [Export]       │ │
│ └────────────────────────────┴───────────────────────────────┘ │
│                                                                  │
│ Forest Statistics                                              │
│ • Total Forest Area: 45,230 ha                                 │
│ • FRA Recognized Area: 8,450 ha (18.7%)                        │
│ • Pending Claims overlapping Forest: 234 (2,340 ha)           │
│                                                                  │
│ [Generate Forest Report] [Download Conflict Data]              │
└─────────────────────────────────────────────────────────────────┘
```

#### 6. Tribal Welfare Officer Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│ FRA Atlas | CSS Integration Dashboard    [User: TWO Bhopal]    │
├─────────────────────────────────────────────────────────────────┤
│ [Beneficiaries] [Scheme Matching] [Enrollment] [Analytics]     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Scheme Recommendations (AI-Powered)                  [2,345]   │
│ ┌──────────────────────────────────────────────────────────────┐│
│ │ Top Priority Beneficiaries                                   ││
│ │                                                              ││
│ │ Ramesh Kumar (MP/IFR/2024/12345)                     Score: 92││
│ │ Village: Bhopal Nagar | Area: 2.5 ha                        ││
│ │ Recommended Schemes:                                         ││
│ │ • PM-KISAN: Eligible ✓ (Not enrolled)                       ││
│ │ • MGNREGA: Active job card                                  ││
│ │ • Pradhan Mantri Awas Yojana: High priority (no house)      ││
│ │ • Van Dhan Vikas Yojana: Suitable (forest produce)          ││
│ │                                                              ││
│ │ [Enroll in Schemes] [View Profile] [Contact]                ││
│ │ ─────────────────────────────────────────────────────────────││
│ │ Sunita Devi (MP/CFR/2024/12350)                      Score: 88││
│ │ ...                                                          ││
│ └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│ ┌────────────────────────────┬───────────────────────────────┐ │
│ │ Scheme Coverage Map        │ Enrollment Statistics         │ │
│ │                            │                               │ │
│ │ [Choropleth: % enrolled]   │ [Bar Chart]                   │ │
│ │ By village                 │ Scheme-wise enrollment        │ │
│ │                            │ • PM-KISAN: 65%               │ │
│ │                            │ • MGNREGA: 82%                │ │
│ │                            │ • PMAY: 45%                   │ │
│ │                            │ • Van Dhan: 23%               │ │
│ └────────────────────────────┴───────────────────────────────┘ │
│                                                                  │
│ Gap Analysis                                                   │
│ • 2,345 FRA beneficiaries not enrolled in PM-KISAN             │
│ • 1,890 eligible for PMAY but not applied                      │
│ • 567 high-priority for livelihood schemes                     │
│                                                                  │
│ [Bulk Enrollment] [Export Beneficiary List] [Generate Report]  │
└─────────────────────────────────────────────────────────────────┘
```

## 5.3 Interactive FRA Atlas

### Atlas Features

```
┌─────────────────────────────────────────────────────────────────┐
│ FRA Atlas - Interactive Map Viewer                              │
├─────────────────────────────────────────────────────────────────┤
│ ┌─────┐                                                         │
│ │Layer│ State: [Madhya Pradesh ▼]  District: [All ▼]           │
│ │Panel│                                                         │
│ │     │ Base Maps                                              │
│ │☑    │ ○ Satellite  ● Streets  ○ Terrain                      │
│ │FRA  │                                                         │
│ │☑ IFR│ FRA Layers                        [Opacity: ████░░]    │
│ │☑ CFR│ ☑ Individual Forest Rights (IFR)  [Style ▼]            │
│ │☑ CR │ ☑ Community Forest Rights (CFR)   [Style ▼]            │
│ │     │ ☑ Community Rights (CR)           [Style ▼]            │
│ │Admin│                                                         │
│ │☐Dist│ Administrative Boundaries                              │
│ │☑ Vil│ ☐ State   ☐ District   ☑ Block   ☑ Village             │
│ │     │                                                         │
│ │Forest Administrative Layers                                  │
│ │☑ For│ ☑ Forest Boundaries                                    │
│ │☐ PA │ ☐ Protected Areas                                      │
│ │☐ WLS│ ☐ Wildlife Sanctuaries                                 │
│ │     │                                                         │
│ │Asset│ Asset Layers (From AI Detection)                       │
│ │☐ Wat│ ☐ Water Bodies         Last updated: 15 Mar 2024       │
│ │☐ Agr│ ☐ Agricultural Land    Last updated: 20 Feb 2024       │
│ │☐ Inf│ ☐ Infrastructure       Last updated: 10 Jan 2024       │
│ │     │                                                         │
│ │Other│ Other Layers                                           │
│ │☐ Rev│ ☐ Revenue Boundaries                                   │
│ │☐ CSS│ ☐ CSS Scheme Coverage                                  │
│ │☐ Dis│ ☑ Disputes & Conflicts  [Show: All ▼]                  │
│ │     │                                                         │
│ │     │ [Reset] [Save View]                                    │
│ └─────┘                                                         │
│         ┌────────────────────────────────────────────────────┐ │
│         │                                                    │ │
│         │           [INTERACTIVE MAP CANVAS]                 │ │
│         │                                                    │ │
│         │  Features:                                         │ │
│         │  • Pan & Zoom (Mouse/Touch)                        │ │
│         │  • Click on features for popup info                │ │
│         │  • Search bar (top-right)                          │ │
│         │  • Measure tool                                    │ │
│         │  • Drawing tool (authorized users)                 │ │
│         │  • Export view as PNG/PDF                          │ │
│         │                                                    │ │
│         │  [+] [-] [⌖] [⤢] [📏] [✏] [💾] [🖨]              │ │
│         │                                                    │ │
│         └────────────────────────────────────────────────────┘ │
│                                                                  │
│ ┌──────────────────────────────────────────────────────────────┐│
│ │ Timeline Slider                                      2024 ▼  ││
│ │ ├───────────────────────●──────────────────────────────────┤ ││
│ │ 2020                   2024                            2026  ││
│ │ [◄] [❚❚] [►]    Speed: 1x ▼       Show: Approval dates      ││
│ └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│ Legend:                                                         │
│ ■ IFR (Approved)  ■ CFR (Approved)  ■ CR (Approved)            │
│ ░ Pending         ░ Under Review    ⚠ Conflict                 │
│                                                                  │
│ [Export Current View] [Share Link] [Print Map]                 │
└─────────────────────────────────────────────────────────────────┘

Popup on Click (Claim Feature):
┌──────────────────────────────────┐
│ IFR Claim: MP/IFR/2024/12345     │
├──────────────────────────────────┤
│ Patta Holder: Ramesh Kumar       │
│ Village: Bhopal Nagar            │
│ Area: 2.5 hectares               │
│ Status: ✓ Approved               │
│ Approval Date: 15 Mar 2024       │
│                                  │
│ [View Full Details]              │
│ [View Document]                  │
│ [Export Record]                  │
└──────────────────────────────────┘
```

### Map Implementation

```typescript
// components/map/MapViewer.tsx

import { useEffect, useRef, useState } from 'react';
import maplibregl from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css';

export function MapViewer() {
  const mapContainer = useRef<HTMLDivElement>(null);
  const map = useRef<maplibregl.Map | null>(null);
  const [activeLayers, setActiveLayers] = useState<string[]>(['ifr', 'cfr', 'villages']);

  useEffect(() => {
    if (!mapContainer.current) return;

    map.current = new maplibregl.Map({
      container: mapContainer.current,
      style: {
        version: 8,
        sources: {
          'osm': {
            type: 'raster',
            tiles: ['https://tile.openstreetmap.org/{z}/{x}/{y}.png'],
            tileSize: 256
          },
          'fra-claims': {
            type: 'vector',
            tiles: [`${process.env.NEXT_PUBLIC_TILE_SERVER_URL}/public.spatial_layers/{z}/{x}/{y}.pbf`],
            minzoom: 6,
            maxzoom: 14
          }
        },
        layers: [
          {
            id: 'osm-tiles',
            type: 'raster',
            source: 'osm'
          },
          {
            id: 'ifr-layer',
            type: 'fill',
            source: 'fra-claims',
            'source-layer': 'spatial_layers',
            filter: ['==', 'layer_type', 'fra_ifr'],
            paint: {
              'fill-color': [
                'match',
                ['get', 'status'],
                'approved', '#10b981',
                'pending', '#fbbf24',
                'rejected', '#ef4444',
                '#6b7280'
              ],
              'fill-opacity': 0.6
            }
          },
          // More layers...
        ]
      },
      center: [78.9629, 20.5937], // Center of India
      zoom: 5
    });

    // Add controls
    map.current.addControl(new maplibregl.NavigationControl());
    map.current.addControl(new maplibregl.FullscreenControl());

    // Add click handler
    map.current.on('click', 'ifr-layer', (e) => {
      if (!e.features || e.features.length === 0) return;

      const feature = e.features[0];
      const popup = new maplibregl.Popup()
        .setLngLat(e.lngLat)
        .setHTML(generatePopupHTML(feature.properties))
        .addTo(map.current!);
    });

    return () => {
      map.current?.remove();
    };
  }, []);

  return (
    <div className="relative w-full h-full">
      <div ref={mapContainer} className="absolute inset-0" />
      <LayerControl
        activeLayers={activeLayers}
        onLayerToggle={(layerId) => {
          map.current?.setLayoutProperty(
            layerId,
            'visibility',
            activeLayers.includes(layerId) ? 'none' : 'visible'
          );
        }}
      />
    </div>
  );
}
```

## 5.4 Village Profile Page

```
┌─────────────────────────────────────────────────────────────────┐
│ Village Profile: Bhopal Nagar                   [Export PDF]    │
│ Block: Berasia | District: Bhopal | State: Madhya Pradesh       │
├─────────────────────────────────────────────────────────────────┤
│ [Overview] [FRA Status] [Demographics] [Schemes] [Assets] [Map] │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Quick Stats                                                     │
│ ┌──────────────┬──────────────┬──────────────┬──────────────┐  │
│ │ Population   │ FRA Claims   │ Area Titled  │ Households   │  │
│ │ 2,345        │ 234          │ 145.5 ha     │ 456          │  │
│ └──────────────┴──────────────┴──────────────┴──────────────┘  │
│                                                                  │
│ ┌────────────────────────────┬───────────────────────────────┐ │
│ │ Village Boundary Map       │ FRA Claims Summary            │ │
│ │                            │                               │ │
│ │ [Map with village outline] │ Claim Type Breakdown:         │ │
│ │ • FRA claim polygons       │ • IFR: 189 (80.8%)            │ │
│ │ • Forest boundary          │ • CFR: 38 (16.2%)             │ │
│ │ • Water bodies             │ • CR: 7 (3.0%)                │ │
│ │ • Infrastructure           │                               │ │
│ │                            │ Status:                       │ │
│ │ [Download KML]             │ • Approved: 215 (91.9%)       │ │
│ │                            │ • Pending: 15 (6.4%)          │ │
│ │                            │ • Rejected: 4 (1.7%)          │ │
│ │                            │                               │ │
│ │                            │ [View All Claims]             │ │
│ └────────────────────────────┴───────────────────────────────┘ │
│                                                                  │
│ CSS Scheme Coverage                                            │
│ ┌────────────────────────┬─────────┬──────────┬──────────────┐ │
│ │ Scheme                 │ Eligible│ Enrolled │ Coverage     │ │
│ ├────────────────────────┼─────────┼──────────┼──────────────┤ │
│ │ PM-KISAN               │ 234     │ 198      │ █████████ 85%│ │
│ │ MGNREGA                │ 456     │ 402      │ █████████ 88%│ │
│ │ PMAY                   │ 156     │ 89       │ █████░░░░ 57%│ │
│ │ Van Dhan Vikas Yojana  │ 234     │ 67       │ ███░░░░░░ 29%│ │
│ └────────────────────────┴─────────┴──────────┴──────────────┘ │
│                                                                  │
│ Infrastructure & Assets (AI-Detected)                          │
│ • Water Bodies: 3 ponds, 1 stream                              │
│ • Agricultural Land: 245 ha (detected from satellite)          │
│ • Roads: 12 km (including 3 km paved)                          │
│ • Schools: 1 primary, 0 secondary                              │
│ • Health Centers: 0 (nearest: 8 km)                            │
│                                                                  │
│ Development Gap Index: 45/100 (Medium Priority)                │
│ Recommended Interventions:                                     │
│ 1. Van Dhan Vikas enrollment drive (low coverage)              │
│ 2. PMAY targeting for 67 households                            │
│ 3. Health sub-center establishment                             │
│                                                                  │
│ [Generate Village Report] [Compare with Similar Villages]      │
└─────────────────────────────────────────────────────────────────┘
```

## 5.5 Design System

### Color Palette (Government-Grade, NOT Purple)

```css
/* Primary: Deep Blue (Authority, Trust) */
--primary-50: #eff6ff;
--primary-100: #dbeafe;
--primary-500: #3b82f6;
--primary-600: #2563eb;
--primary-700: #1d4ed8;
--primary-900: #1e3a8a;

/* Secondary: Forest Green (FRA Theme) */
--secondary-50: #f0fdf4;
--secondary-100: #dcfce7;
--secondary-500: #22c55e;
--secondary-600: #16a34a;
--secondary-700: #15803d;

/* Accent: Saffron (National Identity) */
--accent-50: #fff7ed;
--accent-100: #ffedd5;
--accent-500: #f97316;
--accent-600: #ea580c;

/* Neutral: Grays */
--neutral-50: #f9fafb;
--neutral-100: #f3f4f6;
--neutral-500: #6b7280;
--neutral-700: #374151;
--neutral-900: #111827;

/* Semantic Colors */
--success: #22c55e;
--warning: #f59e0b;
--error: #ef4444;
--info: #3b82f6;
```

### Typography

```css
/* Font Family */
--font-primary: 'Inter', 'Noto Sans Devanagari', sans-serif;
--font-display: 'Poppins', 'Noto Sans', sans-serif;

/* Font Sizes */
--text-xs: 0.75rem;    /* 12px */
--text-sm: 0.875rem;   /* 14px */
--text-base: 1rem;     /* 16px */
--text-lg: 1.125rem;   /* 18px */
--text-xl: 1.25rem;    /* 20px */
--text-2xl: 1.5rem;    /* 24px */
--text-3xl: 1.875rem;  /* 30px */
--text-4xl: 2.25rem;   /* 36px */

/* Line Heights */
--leading-body: 1.5;     /* 150% */
--leading-heading: 1.2;  /* 120% */

/* Font Weights */
--font-normal: 400;
--font-medium: 500;
--font-semibold: 600;
```

### Spacing System (8px base)

```css
--space-1: 0.5rem;   /* 8px */
--space-2: 1rem;     /* 16px */
--space-3: 1.5rem;   /* 24px */
--space-4: 2rem;     /* 32px */
--space-6: 3rem;     /* 48px */
--space-8: 4rem;     /* 64px */
```

### Component Examples

#### Official Header

```
┌─────────────────────────────────────────────────────────────────┐
│ [National Emblem]  FRA Atlas & WebGIS DSS                       │
│ Ministry of Tribal Affairs, Government of India                 │
│                                                      [User Menu] │
└─────────────────────────────────────────────────────────────────┘
```

#### Dark Mode

- Toggle available in user menu
- Preserves readability and accessibility
- Inverted color scheme with proper contrast

#### Multilingual Support

```typescript
// i18n configuration
{
  "en": {
    "dashboard.title": "FRA Atlas Dashboard",
    "claims.total": "Total Claims"
  },
  "hi": {
    "dashboard.title": "FRA एटलस डैशबोर्ड",
    "claims.total": "कुल दावे"
  },
  "or": {
    "dashboard.title": "FRA ଆଟଲାସ୍ ଡ୍ୟାସବୋର୍ଡ",
    "claims.total": "ମୋଟ ଦାବି"
  }
}
```

---

# SECTION 6: DECISION SUPPORT SYSTEM (DSS)

## 6.1 DSS Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         DSS ENGINE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐ │
│  │  Rule Engine     │  │  Scoring Engine  │  │  ML Predictor │ │
│  │  (Eligibility)   │  │  (Prioritization)│  │  (Optional)   │ │
│  └──────────────────┘  └──────────────────┘  └───────────────┘ │
│           │                     │                     │          │
│           └─────────────────────┴─────────────────────┘          │
│                                 │                                │
│                    ┌────────────▼────────────┐                   │
│                    │  Recommendation Engine  │                   │
│                    └────────────┬────────────┘                   │
│                                 │                                │
│           ┌─────────────────────┼─────────────────────┐          │
│           │                     │                     │          │
│    ┌──────▼──────┐      ┌──────▼──────┐      ┌──────▼──────┐   │
│    │  Scheme     │      │  Village    │      │  Policy     │   │
│    │  Matching   │      │  Develop.   │      │  Simulation │   │
│    └─────────────┘      └─────────────┘      └─────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 6.2 Rule-Based Engine for CSS Eligibility

### Scheme Rules Database

```sql
CREATE TABLE css_schemes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scheme_code VARCHAR(50) UNIQUE NOT NULL,
    scheme_name VARCHAR(255) NOT NULL,
    scheme_name_hindi VARCHAR(255),
    ministry VARCHAR(255),
    description TEXT,
    eligibility_rules JSONB NOT NULL,
    benefits JSONB,
    application_process TEXT,
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Example scheme entry
INSERT INTO css_schemes (scheme_code, scheme_name, eligibility_rules, benefits) VALUES
('PM_KISAN', 'Pradhan Mantri Kisan Samman Nidhi',
 '{
   "rules": [
     {"field": "has_agricultural_land", "operator": "equals", "value": true},
     {"field": "land_area_hectares", "operator": "less_than", "value": 2},
     {"field": "claim_type", "operator": "in", "value": ["IFR", "CR"]}
   ],
   "logic": "AND"
 }',
 '{
   "amount_per_year": 6000,
   "disbursement": "3 installments",
   "currency": "INR"
 }');
```

### Eligibility Matching Logic

```python
# dss/eligibility_matcher.py

class EligibilityMatcher:
    def __init__(self, db_client):
        self.db = db_client
        self.schemes = self.load_schemes()

    def load_schemes(self):
        return self.db.table('css_schemes').select('*').eq('active', True).execute().data

    def match_beneficiary(self, beneficiary_data):
        """
        Match beneficiary against all scheme eligibility rules
        Returns list of eligible schemes with match confidence
        """
        matched_schemes = []

        for scheme in self.schemes:
            eligibility = self.check_eligibility(
                beneficiary_data,
                scheme['eligibility_rules']
            )

            if eligibility['eligible']:
                matched_schemes.append({
                    'scheme_code': scheme['scheme_code'],
                    'scheme_name': scheme['scheme_name'],
                    'confidence': eligibility['confidence'],
                    'missing_docs': eligibility['missing_documents'],
                    'benefits': scheme['benefits']
                })

        return matched_schemes

    def check_eligibility(self, beneficiary, rules):
        """
        Evaluate rules against beneficiary data
        """
        rule_results = []
        missing_docs = []

        for rule in rules['rules']:
            field = rule['field']
            operator = rule['operator']
            expected_value = rule['value']

            actual_value = beneficiary.get(field)

            if actual_value is None:
                missing_docs.append(field)
                rule_results.append(False)
                continue

            result = self.evaluate_rule(actual_value, operator, expected_value)
            rule_results.append(result)

        # Apply logic (AND/OR)
        if rules['logic'] == 'AND':
            eligible = all(rule_results)
        else:
            eligible = any(rule_results)

        # Calculate confidence based on missing docs
        confidence = (len([r for r in rule_results if r]) / len(rule_results)) * 100

        return {
            'eligible': eligible,
            'confidence': confidence,
            'missing_documents': missing_docs
        }

    def evaluate_rule(self, actual, operator, expected):
        if operator == 'equals':
            return actual == expected
        elif operator == 'less_than':
            return actual < expected
        elif operator == 'greater_than':
            return actual > expected
        elif operator == 'in':
            return actual in expected
        elif operator == 'not_in':
            return actual not in expected
        return False
```

## 6.3 Scoring Algorithms

### Composite Priority Score

```python
# dss/scoring_engine.py

class ScoringEngine:
    def calculate_priority_score(self, beneficiary_id):
        """
        Calculate multi-dimensional priority score for beneficiary
        Range: 0-100 (higher = more urgent)
        """
        scores = {}

        # 1. Economic Vulnerability (30 points)
        scores['economic'] = self.calculate_economic_score(beneficiary_id)

        # 2. Infrastructure Deficit (25 points)
        scores['infrastructure'] = self.calculate_infrastructure_score(beneficiary_id)

        # 3. Resource Dependency (20 points)
        scores['resource_dependency'] = self.calculate_resource_score(beneficiary_id)

        # 4. Geographic Remoteness (15 points)
        scores['remoteness'] = self.calculate_remoteness_score(beneficiary_id)

        # 5. Scheme Coverage Gap (10 points)
        scores['coverage_gap'] = self.calculate_coverage_gap(beneficiary_id)

        total_score = sum(scores.values())

        return {
            'total_score': total_score,
            'breakdown': scores,
            'priority_band': self.get_priority_band(total_score)
        }

    def calculate_economic_score(self, beneficiary_id):
        # Factors: land size, income sources, BPL status
        data = self.get_beneficiary_data(beneficiary_id)

        score = 0

        # Small landholding = higher score
        if data['land_area_hectares'] < 1:
            score += 15
        elif data['land_area_hectares'] < 2:
            score += 10
        else:
            score += 5

        # BPL status
        if data.get('bpl_card', False):
            score += 10

        # No other income source
        if not data.get('has_other_income', False):
            score += 5

        return min(score, 30)

    def calculate_infrastructure_score(self, beneficiary_id):
        # Factors: distance to road, school, health center, water source
        village_data = self.get_village_data(beneficiary_id)

        score = 0

        # Distance to paved road
        if village_data['distance_to_road_km'] > 10:
            score += 8
        elif village_data['distance_to_road_km'] > 5:
            score += 5

        # No health center in village
        if not village_data['has_health_center']:
            score += 7

        # No school
        if not village_data['has_school']:
            score += 5

        # Water scarcity
        if village_data.get('water_stress_index', 0) > 0.7:
            score += 5

        return min(score, 25)

    def calculate_resource_score(self, beneficiary_id):
        # Forest resource dependency
        data = self.get_beneficiary_data(beneficiary_id)

        score = 0

        # High dependency on forest produce
        if data.get('forest_income_percentage', 0) > 50:
            score += 15
        elif data.get('forest_income_percentage', 0) > 30:
            score += 10

        # Collects NTFP
        if data.get('collects_ntfp', False):
            score += 5

        return min(score, 20)

    def get_priority_band(self, score):
        if score >= 80:
            return 'Critical'
        elif score >= 60:
            return 'High'
        elif score >= 40:
            return 'Medium'
        else:
            return 'Low'
```

### Village Development Gap Index

```python
def calculate_village_gap_index(village_id):
    """
    Multi-dimensional gap index for village-level development
    Range: 0-100 (higher = larger gap)
    """

    village = get_village_data(village_id)

    # Infrastructure gaps (0-40 points)
    infrastructure_gap = 0
    infrastructure_gap += 10 if not village['has_paved_road'] else 0
    infrastructure_gap += 10 if not village['has_electricity'] else 0
    infrastructure_gap += 10 if not village['has_health_center'] else 0
    infrastructure_gap += 10 if not village['has_secondary_school'] else 0

    # Scheme coverage gap (0-30 points)
    avg_scheme_coverage = calculate_avg_scheme_coverage(village_id)
    coverage_gap = (1 - avg_scheme_coverage) * 30

    # Resource access gap (0-30 points)
    resource_gap = 0
    resource_gap += 15 if village['water_stress_index'] > 0.7 else 0
    resource_gap += 10 if village['forest_distance_km'] > 5 else 0
    resource_gap += 5 if village['market_distance_km'] > 20 else 0

    total_gap = infrastructure_gap + coverage_gap + resource_gap

    return {
        'gap_index': total_gap,
        'infrastructure_gap': infrastructure_gap,
        'coverage_gap': coverage_gap,
        'resource_gap': resource_gap,
        'priority': 'High' if total_gap > 60 else 'Medium' if total_gap > 30 else 'Low'
    }
```

## 6.4 Scheme Recommendation Dashboard

### UI Design

```
┌─────────────────────────────────────────────────────────────────┐
│ DSS Recommendations - Bhopal District                           │
├─────────────────────────────────────────────────────────────────┤
│ [Individual] [Village-Level] [Scheme-wise] [Gap Analysis]      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Top Priority Beneficiaries                          [Export]    │
│ ┌──────────────────────────────────────────────────────────────┐│
│ │ [Filter: Priority ▼] [Village ▼] [Scheme ▼]   [Search...]   ││
│ │                                                              ││
│ │ Name: Ramesh Kumar                          Priority: 92/100││
│ │ Village: Bhopal Nagar | Claim: MP/IFR/2024/12345            ││
│ │                                                              ││
│ │ Priority Breakdown:                                          ││
│ │ • Economic Vulnerability: ████████████ 28/30                ││
│ │ • Infrastructure Deficit: ███████████░ 22/25                ││
│ │ • Resource Dependency:    ████████░░░░ 16/20                ││
│ │ • Remoteness:             ██████████░░ 12/15                ││
│ │ • Coverage Gap:           ████████████ 10/10                ││
│ │                                                              ││
│ │ Recommended Schemes (5):                                     ││
│ │ 1. ✓ PM-KISAN (Eligible, Not Enrolled) - ₹6,000/year       ││
│ │    [Enroll Now]                                             ││
│ │ 2. ✓ PMAY (Eligible, Pending Docs) - House construction    ││
│ │    Missing: Income certificate  [Upload]                    ││
│ │ 3. ✓ Van Dhan Vikas (High Match) - Livelihood support      ││
│ │    [Schedule Training]                                      ││
│ │ 4. ~ MGNREGA (Already enrolled) - Job card active           ││
│ │ 5. ✓ Ujjwala (Eligible) - LPG connection                   ││
│ │    [Apply]                                                  ││
│ │                                                              ││
│ │ Estimated Annual Benefit if Enrolled: ₹28,450               ││
│ │                                                              ││
│ │ [View Full Profile] [Bulk Actions ▼]                        ││
│ └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│ Scheme-wise Gap Analysis                                       │
│ ┌────────────────────┬─────────┬──────────┬─────────────────┐  │
│ │ Scheme             │ Eligible│ Enrolled │ Gap             │  │
│ ├────────────────────┼─────────┼──────────┼─────────────────┤  │
│ │ PM-KISAN           │ 12,345  │ 8,234    │ 4,111 (33%)     │  │
│ │ PMAY               │ 8,900   │ 4,200    │ 4,700 (53%) ⚠  │  │
│ │ Van Dhan Vikas     │ 9,500   │ 2,100    │ 7,400 (78%) 🔴 │  │
│ │ Ujjwala            │ 7,800   │ 6,100    │ 1,700 (22%)     │  │
│ └────────────────────┴─────────┴──────────┴─────────────────┘  │
│                                                                  │
│ [Generate Campaign Plan] [Bulk Enrollment Tool]                │
└─────────────────────────────────────────────────────────────────┘
```

## 6.5 Policy Simulation Tool

### Interface

```
┌─────────────────────────────────────────────────────────────────┐
│ Policy Simulation Tool                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Simulation Parameters                                           │
│                                                                  │
│ Scenario Name: [Van Dhan Expansion - Bhopal District]          │
│                                                                  │
│ Target Population:                                              │
│ ○ All FRA beneficiaries  ○ Specific villages  ● Custom filter  │
│                                                                  │
│ Filter Criteria:                                                │
│ • District: [Bhopal ▼]                                          │
│ • Forest income >30%: [✓]                                       │
│ • Not enrolled in Van Dhan: [✓]                                 │
│ • Land area <2 ha: [✓]                                          │
│                                                                  │
│ Matched Beneficiaries: 3,456                                    │
│                                                                  │
│ Intervention:                                                   │
│ Scheme: [Van Dhan Vikas Yojana ▼]                               │
│ Enrollment Target: [80%] of matched beneficiaries               │
│ Estimated Cost per Beneficiary: [₹15,000]                       │
│                                                                  │
│ [Run Simulation]                                                │
│                                                                  │
│ ─────────────────────────────────────────────────────────────── │
│                                                                  │
│ Simulation Results                                              │
│                                                                  │
│ Impact Projections:                                             │
│ • Beneficiaries Covered: 2,765 (80% of 3,456)                  │
│ • Estimated Annual Income Increase: ₹12,000/family             │
│ • Total Budget Required: ₹4.15 Crores                           │
│ • Implementation Timeline: 6 months                             │
│                                                                  │
│ Geographic Distribution:                                        │
│ [Map showing affected villages with coverage %]                │
│                                                                  │
│ Village-wise Breakdown:                                         │
│ ┌──────────────────┬────────────┬──────────┬────────────────┐  │
│ │ Village          │ Eligible   │ Target   │ Budget (Lakhs) │  │
│ ├──────────────────┼────────────┼──────────┼────────────────┤  │
│ │ Bhopal Nagar     │ 234        │ 187      │ 28.05          │  │
│ │ Indore           │ 456        │ 365      │ 54.75          │  │
│ │ ...              │            │          │                │  │
│ └──────────────────┴────────────┴──────────┴────────────────┘  │
│                                                                  │
│ Expected Outcomes:                                              │
│ • Reduction in forest dependency: -15%                          │
│ • Income diversification: +35%                                  │
│ • Women SHG participation: +450 members                         │
│                                                                  │
│ [Export Report] [Save Scenario] [Compare with Baseline]        │
└─────────────────────────────────────────────────────────────────┘
```

---

# SECTION 7: DATABASE DESIGN

## 7.1 Complete Database Schema

```sql
-- ============================================================================
-- FRA ATLAS & WebGIS DSS - DATABASE SCHEMA
-- ============================================================================

-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ============================================================================
-- CORE FRA DATA TABLES
-- ============================================================================

-- States master
CREATE TABLE states (
    code VARCHAR(2) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    name_hindi VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT now()
);

INSERT INTO states (code, name, name_hindi) VALUES
('MP', 'Madhya Pradesh', 'मध्य प्रदेश'),
('OD', 'Odisha', 'ओडिशा'),
('TR', 'Tripura', 'त्रिपुरा'),
('TG', 'Telangana', 'तेलंगाना');

-- Administrative hierarchy
CREATE TABLE districts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(4) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    state_code VARCHAR(2) REFERENCES states(code),
    geometry GEOMETRY(MultiPolygon, 4326),
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_districts_state ON districts(state_code);
CREATE INDEX idx_districts_geom ON districts USING GIST(geometry);

CREATE TABLE blocks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(6) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    district_id UUID REFERENCES districts(id),
    geometry GEOMETRY(MultiPolygon, 4326),
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_blocks_district ON blocks(district_id);
CREATE INDEX idx_blocks_geom ON blocks USING GIST(geometry);

CREATE TABLE villages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(10) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    block_id UUID REFERENCES blocks(id),
    population INTEGER,
    total_households INTEGER,
    geometry GEOMETRY(MultiPolygon, 4326),
    centroid GEOMETRY(Point, 4326),

    -- Infrastructure attributes
    has_paved_road BOOLEAN DEFAULT false,
    has_electricity BOOLEAN DEFAULT false,
    has_health_center BOOLEAN DEFAULT false,
    has_primary_school BOOLEAN DEFAULT false,
    has_secondary_school BOOLEAN DEFAULT false,

    -- Distance metrics (km)
    distance_to_road_km DECIMAL(6, 2),
    distance_to_market_km DECIMAL(6, 2),
    distance_to_health_km DECIMAL(6, 2),

    -- Resource metrics
    water_stress_index DECIMAL(3, 2), -- 0-1
    forest_distance_km DECIMAL(6, 2),

    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_villages_block ON villages(block_id);
CREATE INDEX idx_villages_geom ON villages USING GIST(geometry);
CREATE INDEX idx_villages_centroid ON villages USING GIST(centroid);

-- FRA Claims master table
CREATE TABLE fra_claims (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_number VARCHAR(50) UNIQUE NOT NULL,
    state_code VARCHAR(2) REFERENCES states(code),
    district_id UUID REFERENCES districts(id),
    block_id UUID REFERENCES blocks(id),
    village_id UUID REFERENCES villages(id),

    claim_type VARCHAR(10) NOT NULL CHECK (claim_type IN ('IFR', 'CFR', 'CR')),
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'approved', 'rejected', 'under_review', 'partial')),

    land_extent_hectares DECIMAL(10, 4),
    survey_number VARCHAR(50),

    approval_date DATE,
    rejection_date DATE,
    rejection_reason TEXT,

    committee_decision_date DATE,
    sdlc_approval BOOLEAN DEFAULT false,
    dlc_approval BOOLEAN DEFAULT false,

    -- Document references
    original_document_url TEXT,
    patta_document_url TEXT,

    -- Data quality
    confidence_score DECIMAL(5, 2), -- 0-100
    needs_verification BOOLEAN DEFAULT false,
    data_source VARCHAR(50) DEFAULT 'digitized', -- 'digitized', 'api_import', 'manual_entry'

    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    created_by UUID REFERENCES auth.users(id)
);

CREATE INDEX idx_claims_state ON fra_claims(state_code);
CREATE INDEX idx_claims_district ON fra_claims(district_id);
CREATE INDEX idx_claims_village ON fra_claims(village_id);
CREATE INDEX idx_claims_type_status ON fra_claims(claim_type, status);
CREATE INDEX idx_claims_number ON fra_claims(claim_number);

-- Enable RLS
ALTER TABLE fra_claims ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY "Users view own state claims"
ON fra_claims FOR SELECT
TO authenticated
USING (
    state_code = (auth.jwt() -> 'app_metadata' ->> 'state_code')
    OR
    (auth.jwt() -> 'app_metadata' ->> 'role') = 'national_admin'
);

CREATE POLICY "District officers manage own district"
ON fra_claims FOR UPDATE
TO authenticated
USING (
    district_id::text = (auth.jwt() -> 'app_metadata' ->> 'district_id')
    AND
    (auth.jwt() -> 'app_metadata' ->> 'role') IN ('district_collector', 'district_admin')
);

-- Patta holders (beneficiaries)
CREATE TABLE patta_holders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID REFERENCES fra_claims(id) ON DELETE CASCADE,

    name VARCHAR(255) NOT NULL,
    father_name VARCHAR(255),
    mother_name VARCHAR(255),
    gender VARCHAR(10) CHECK (gender IN ('male', 'female', 'other')),
    age INTEGER,

    aadhaar_number VARCHAR(12), -- encrypted in production
    voter_id VARCHAR(20),
    bpl_card BOOLEAN DEFAULT false,

    mobile_number VARCHAR(15),
    email VARCHAR(255),

    -- Economic data
    primary_occupation VARCHAR(100),
    has_other_income BOOLEAN DEFAULT false,
    forest_income_percentage INTEGER CHECK (forest_income_percentage BETWEEN 0 AND 100),
    collects_ntfp BOOLEAN DEFAULT false,

    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_patta_claim ON patta_holders(claim_id);
CREATE INDEX idx_patta_name ON patta_holders(name);

-- CFR claims can have multiple families
CREATE TABLE cfr_families (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID REFERENCES fra_claims(id) ON DELETE CASCADE,
    family_head_name VARCHAR(255) NOT NULL,
    family_size INTEGER,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Spatial data for claims
CREATE TABLE claim_spatial_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID REFERENCES fra_claims(id) ON DELETE CASCADE,

    geometry GEOMETRY(Polygon, 4326),
    centroid GEOMETRY(Point, 4326),
    area_hectares DECIMAL(10, 4),
    perimeter_meters DECIMAL(12, 2),

    boundary_source VARCHAR(50) CHECK (boundary_source IN ('gps_survey', 'digitized', 'estimated', 'satellite')),
    survey_date DATE,
    surveyor_name VARCHAR(255),
    accuracy_meters DECIMAL(6, 2),

    created_at TIMESTAMPTZ DEFAULT now(),
    version INTEGER DEFAULT 1
);

CREATE INDEX idx_claim_spatial_geom ON claim_spatial_data USING GIST(geometry);
CREATE INDEX idx_claim_spatial_claim ON claim_spatial_data(claim_id);

-- Spatial layers (generalized for all polygon data)
CREATE TABLE spatial_layers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    layer_type VARCHAR(50) NOT NULL, -- 'fra_ifr', 'fra_cfr', 'fra_cr', 'admin_district', 'forest_reserve', 'protected_area', 'water_body', etc.
    state_code VARCHAR(2) REFERENCES states(code),
    name VARCHAR(255) NOT NULL,

    geometry GEOMETRY(Geometry, 4326) NOT NULL,
    centroid GEOMETRY(Point, 4326),
    area_hectares DECIMAL(12, 4),

    properties JSONB, -- Flexible attributes
    style JSONB, -- Map styling

    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    created_by UUID REFERENCES auth.users(id),
    version INTEGER DEFAULT 1
);

CREATE INDEX idx_spatial_layers_geom ON spatial_layers USING GIST(geometry);
CREATE INDEX idx_spatial_layers_type_state ON spatial_layers(layer_type, state_code);
CREATE INDEX idx_spatial_layers_properties ON spatial_layers USING GIN(properties);

-- ============================================================================
-- DOCUMENT PROCESSING TABLES
-- ============================================================================

-- Document processing jobs
CREATE TABLE document_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    state_code VARCHAR(2) REFERENCES states(code),
    district_id UUID REFERENCES districts(id),
    village_id UUID REFERENCES villages(id),

    original_filename VARCHAR(255) NOT NULL,
    document_url TEXT NOT NULL,
    document_type VARCHAR(50) NOT NULL,
    file_size_bytes BIGINT,

    status VARCHAR(50) DEFAULT 'queued'
        CHECK (status IN ('queued', 'processing', 'extraction_complete', 'validated',
                          'needs_review', 'approved', 'failed')),

    -- Processing results
    extraction_result_url TEXT,
    confidence_overall DECIMAL(5, 2),

    error_message TEXT,
    review_notes TEXT,

    uploaded_by UUID REFERENCES auth.users(id),
    reviewed_by UUID REFERENCES auth.users(id),

    created_at TIMESTAMPTZ DEFAULT now(),
    completed_at TIMESTAMPTZ
);

CREATE INDEX idx_doc_jobs_status ON document_jobs(status);
CREATE INDEX idx_doc_jobs_state ON document_jobs(state_code);
CREATE INDEX idx_doc_jobs_uploaded_by ON document_jobs(uploaded_by);

-- Extracted entities (before final approval)
CREATE TABLE extracted_entities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID REFERENCES document_jobs(id) ON DELETE CASCADE,

    entity_type VARCHAR(50) NOT NULL, -- 'claim_number', 'patta_holder', 'village', etc.
    raw_value TEXT,
    normalized_value TEXT,
    confidence DECIMAL(5, 2),

    bounding_box JSONB, -- {x, y, width, height} on document

    validation_status VARCHAR(20) CHECK (validation_status IN ('pending', 'valid', 'invalid', 'corrected')),
    corrected_value TEXT,
    corrected_by UUID REFERENCES auth.users(id),

    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_extracted_entities_job ON extracted_entities(job_id);
CREATE INDEX idx_extracted_entities_type ON extracted_entities(entity_type);

-- ============================================================================
-- DISPUTES & CONFLICTS
-- ============================================================================

CREATE TABLE disputes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dispute_type VARCHAR(50) NOT NULL
        CHECK (dispute_type IN ('boundary_overlap', 'protected_area_conflict',
                                'revenue_conflict', 'documentation', 'administrative')),
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),

    claim_id UUID REFERENCES fra_claims(id),
    conflicting_claim_id UUID REFERENCES fra_claims(id),

    description TEXT NOT NULL,
    geometry GEOMETRY(Geometry, 4326), -- disputed boundary

    status VARCHAR(50) DEFAULT 'open'
        CHECK (status IN ('open', 'under_review', 'field_verification_pending', 'resolved', 'escalated')),

    assigned_to UUID REFERENCES auth.users(id),
    resolution_notes TEXT,
    resolution_date DATE,

    created_at TIMESTAMPTZ DEFAULT now(),
    created_by UUID REFERENCES auth.users(id)
);

CREATE INDEX idx_disputes_claim ON disputes(claim_id);
CREATE INDEX idx_disputes_status ON disputes(status);
CREATE INDEX idx_disputes_severity ON disputes(severity);

-- ============================================================================
-- CSS SCHEMES & DSS
-- ============================================================================

-- Centrally Sponsored Schemes master
CREATE TABLE css_schemes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scheme_code VARCHAR(50) UNIQUE NOT NULL,
    scheme_name VARCHAR(255) NOT NULL,
    scheme_name_hindi VARCHAR(255),
    scheme_name_regional JSONB, -- {or: "", te: ""}

    ministry VARCHAR(255),
    description TEXT,
    eligibility_rules JSONB NOT NULL,
    benefits JSONB,
    application_process TEXT,

    active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Scheme enrollment tracking
CREATE TABLE scheme_enrollments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patta_holder_id UUID REFERENCES patta_holders(id),
    scheme_id UUID REFERENCES css_schemes(id),

    enrollment_date DATE,
    enrollment_status VARCHAR(50) CHECK (enrollment_status IN ('enrolled', 'pending', 'rejected', 'inactive')),

    application_number VARCHAR(100),
    benefit_received DECIMAL(12, 2),
    last_benefit_date DATE,

    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_enrollments_holder ON scheme_enrollments(patta_holder_id);
CREATE INDEX idx_enrollments_scheme ON scheme_enrollments(scheme_id);

-- DSS Recommendations
CREATE TABLE dss_recommendations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patta_holder_id UUID REFERENCES patta_holders(id),

    priority_score DECIMAL(5, 2), -- 0-100
    priority_band VARCHAR(20) CHECK (priority_band IN ('low', 'medium', 'high', 'critical')),

    score_breakdown JSONB, -- {economic: 28, infrastructure: 22, ...}

    recommended_schemes JSONB, -- [{scheme_code, confidence, reason}, ...]

    generated_at TIMESTAMPTZ DEFAULT now(),
    expires_at TIMESTAMPTZ DEFAULT (now() + INTERVAL '30 days')
);

CREATE INDEX idx_recommendations_holder ON dss_recommendations(patta_holder_id);
CREATE INDEX idx_recommendations_score ON dss_recommendations(priority_score DESC);

-- Village development metrics
CREATE TABLE village_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    village_id UUID REFERENCES villages(id),

    gap_index DECIMAL(5, 2), -- 0-100
    infrastructure_gap DECIMAL(5, 2),
    coverage_gap DECIMAL(5, 2),
    resource_gap DECIMAL(5, 2),

    avg_scheme_coverage DECIMAL(5, 4), -- 0-1

    calculated_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_village_metrics_village ON village_metrics(village_id);
CREATE INDEX idx_village_metrics_gap ON village_metrics(gap_index DESC);

-- ============================================================================
-- ASSET MAPPING (FUTURE)
-- ============================================================================

CREATE TABLE asset_mapping_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    state_code VARCHAR(2) REFERENCES states(code),

    bbox GEOMETRY(Polygon, 4326),
    asset_types VARCHAR[] NOT NULL,

    imagery_source VARCHAR(50), -- 'sentinel2', 'landsat8'
    imagery_date DATE,

    status VARCHAR(50) DEFAULT 'queued',
    result_url TEXT,
    error_message TEXT,

    created_at TIMESTAMPTZ DEFAULT now(),
    completed_at TIMESTAMPTZ
);

CREATE TABLE detected_assets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID REFERENCES asset_mapping_jobs(id),

    asset_type VARCHAR(50) NOT NULL, -- 'water_body', 'agricultural_land', 'forest', 'infrastructure'
    geometry GEOMETRY(Geometry, 4326) NOT NULL,
    properties JSONB,

    confidence_score DECIMAL(5, 4),
    village_id UUID REFERENCES villages(id),

    detected_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_detected_assets_geom ON detected_assets USING GIST(geometry);
CREATE INDEX idx_detected_assets_type ON detected_assets(asset_type);
CREATE INDEX idx_detected_assets_village ON detected_assets(village_id);

-- ============================================================================
-- AUDIT & LOGS
-- ============================================================================

CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),

    old_values JSONB,
    new_values JSONB,

    user_id UUID REFERENCES auth.users(id),
    user_role VARCHAR(100),
    ip_address INET,

    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_audit_logs_table_record ON audit_logs(table_name, record_id);
CREATE INDEX idx_audit_logs_user ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_created ON audit_logs(created_at DESC);

-- ============================================================================
-- MATERIALIZED VIEWS (for performance)
-- ============================================================================

-- District-level statistics
CREATE MATERIALIZED VIEW mv_district_stats AS
SELECT
    d.id as district_id,
    d.code as district_code,
    d.name as district_name,
    d.state_code,
    COUNT(DISTINCT fc.id) as total_claims,
    COUNT(DISTINCT fc.id) FILTER (WHERE fc.status = 'approved') as approved_claims,
    COUNT(DISTINCT fc.id) FILTER (WHERE fc.status = 'pending') as pending_claims,
    COUNT(DISTINCT fc.id) FILTER (WHERE fc.status = 'rejected') as rejected_claims,
    SUM(fc.land_extent_hectares) FILTER (WHERE fc.status = 'approved') as total_area_hectares,
    COUNT(DISTINCT fc.id) FILTER (WHERE fc.claim_type = 'IFR') as ifr_count,
    COUNT(DISTINCT fc.id) FILTER (WHERE fc.claim_type = 'CFR') as cfr_count,
    COUNT(DISTINCT fc.id) FILTER (WHERE fc.claim_type = 'CR') as cr_count,
    AVG(fc.confidence_score) as avg_confidence,
    COUNT(DISTINCT disp.id) FILTER (WHERE disp.status = 'open') as open_disputes
FROM districts d
LEFT JOIN fra_claims fc ON d.id = fc.district_id
LEFT JOIN disputes disp ON fc.id = disp.claim_id
GROUP BY d.id, d.code, d.name, d.state_code;

CREATE UNIQUE INDEX ON mv_district_stats(district_id);

-- Refresh function
CREATE OR REPLACE FUNCTION refresh_district_stats()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY mv_district_stats;
END;
$$ LANGUAGE plpgsql;

-- Village-level statistics
CREATE MATERIALIZED VIEW mv_village_stats AS
SELECT
    v.id as village_id,
    v.code as village_code,
    v.name as village_name,
    COUNT(DISTINCT fc.id) as total_claims,
    COUNT(DISTINCT fc.id) FILTER (WHERE fc.status = 'approved') as approved_claims,
    SUM(fc.land_extent_hectares) FILTER (WHERE fc.status = 'approved') as total_area_hectares,
    COUNT(DISTINCT ph.id) as total_beneficiaries,
    COUNT(DISTINCT se.id) / NULLIF(COUNT(DISTINCT ph.id), 0)::DECIMAL as avg_schemes_per_beneficiary
FROM villages v
LEFT JOIN fra_claims fc ON v.id = fc.village_id
LEFT JOIN patta_holders ph ON fc.id = ph.claim_id
LEFT JOIN scheme_enrollments se ON ph.id = se.patta_holder_id
GROUP BY v.id, v.code, v.name;

CREATE UNIQUE INDEX ON mv_village_stats(village_id);

-- ============================================================================
-- FUNCTIONS & TRIGGERS
-- ============================================================================

-- Auto-update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER fra_claims_updated_at
BEFORE UPDATE ON fra_claims
FOR EACH ROW
EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER villages_updated_at
BEFORE UPDATE ON villages
FOR EACH ROW
EXECUTE FUNCTION update_updated_at();

-- Audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_logs (table_name, record_id, action, new_values, user_id)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', row_to_json(NEW), auth.uid());
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_logs (table_name, record_id, action, old_values, new_values, user_id)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', row_to_json(OLD), row_to_json(NEW), auth.uid());
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_logs (table_name, record_id, action, old_values, user_id)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', row_to_json(OLD), auth.uid());
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Apply audit to critical tables
CREATE TRIGGER fra_claims_audit
AFTER INSERT OR UPDATE OR DELETE ON fra_claims
FOR EACH ROW EXECUTE FUNCTION audit_trigger();

-- Calculate village gap index function
CREATE OR REPLACE FUNCTION calculate_village_gap_index(v_id UUID)
RETURNS DECIMAL AS $$
DECLARE
    v_data RECORD;
    infrastructure_gap DECIMAL;
    coverage_gap DECIMAL;
    resource_gap DECIMAL;
    total_gap DECIMAL;
BEGIN
    SELECT * INTO v_data FROM villages WHERE id = v_id;

    infrastructure_gap := 0;
    IF NOT v_data.has_paved_road THEN infrastructure_gap := infrastructure_gap + 10; END IF;
    IF NOT v_data.has_electricity THEN infrastructure_gap := infrastructure_gap + 10; END IF;
    IF NOT v_data.has_health_center THEN infrastructure_gap := infrastructure_gap + 10; END IF;
    IF NOT v_data.has_secondary_school THEN infrastructure_gap := infrastructure_gap + 10; END IF;

    -- Calculate scheme coverage gap
    SELECT
        (1 - (COUNT(DISTINCT se.id)::DECIMAL / NULLIF(COUNT(DISTINCT ph.id), 0)::DECIMAL)) * 30
    INTO coverage_gap
    FROM fra_claims fc
    LEFT JOIN patta_holders ph ON fc.id = ph.claim_id
    LEFT JOIN scheme_enrollments se ON ph.id = se.patta_holder_id
    WHERE fc.village_id = v_id;

    resource_gap := 0;
    IF v_data.water_stress_index > 0.7 THEN resource_gap := resource_gap + 15; END IF;
    IF v_data.forest_distance_km > 5 THEN resource_gap := resource_gap + 10; END IF;
    IF v_data.distance_to_market_km > 20 THEN resource_gap := resource_gap + 5; END IF;

    total_gap := infrastructure_gap + COALESCE(coverage_gap, 0) + resource_gap;

    RETURN total_gap;
END;
$$ LANGUAGE plpgsql;
```

---

# SECTION 8: SECURITY & GOVERNANCE

## 8.1 Authentication & Authorization

### Role-Based Access Control (RBAC)

```sql
-- User roles enum
CREATE TYPE user_role AS ENUM (
    'national_admin',
    'state_admin',
    'district_collector',
    'block_officer',
    'forest_officer',
    'tribal_welfare_officer',
    'data_entry_operator',
    'viewer'
);

-- User profiles (extends Supabase auth.users)
CREATE TABLE user_profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    role user_role NOT NULL,

    -- Geographic jurisdiction
    state_code VARCHAR(2) REFERENCES states(code),
    district_id UUID REFERENCES districts(id),
    block_id UUID REFERENCES blocks(id),

    full_name VARCHAR(255) NOT NULL,
    designation VARCHAR(255),
    department VARCHAR(255),
    office_address TEXT,
    phone VARCHAR(15),

    is_active BOOLEAN DEFAULT true,
    last_login TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- RLS Policies by Role
CREATE POLICY "National admins see all data"
ON fra_claims FOR SELECT
TO authenticated
USING ((auth.jwt() -> 'app_metadata' ->> 'role') = 'national_admin');

CREATE POLICY "State admins see own state"
ON fra_claims FOR SELECT
TO authenticated
USING (
    state_code = (auth.jwt() -> 'app_metadata' ->> 'state_code')
    AND (auth.jwt() -> 'app_metadata' ->> 'role') = 'state_admin'
);

CREATE POLICY "District collectors manage own district"
ON fra_claims FOR ALL
TO authenticated
USING (
    district_id::text = (auth.jwt() -> 'app_metadata' ->> 'district_id')
    AND (auth.jwt() -> 'app_metadata' ->> 'role') IN ('district_collector', 'district_admin')
);

CREATE POLICY "Block officers insert own block"
ON document_jobs FOR INSERT
TO authenticated
WITH CHECK (
    block_id::text = (auth.jwt() -> 'app_metadata' ->> 'block_id')
    AND (auth.jwt() -> 'app_metadata' ->> 'role') = 'block_officer'
);
```

### JWT Payload Structure

```json
{
  "sub": "user-uuid",
  "email": "dc.bhopal@mp.gov.in",
  "app_metadata": {
    "role": "district_collector",
    "state_code": "MP",
    "district_id": "uuid",
    "permissions": ["read_claims", "approve_claims", "manage_disputes"]
  },
  "exp": 1709567890
}
```

## 8.2 Data Encryption

### At Rest
- **Database**: AES-256 encryption via Supabase
- **Files**: Server-side encryption in Supabase Storage
- **Sensitive Fields**: Application-level encryption for PII

```typescript
// Encrypt sensitive data before storage
import crypto from 'crypto';

const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY; // 32-byte key

function encrypt(text: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-cbc', Buffer.from(ENCRYPTION_KEY), iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  return iv.toString('hex') + ':' + encrypted;
}

function decrypt(text: string): string {
  const parts = text.split(':');
  const iv = Buffer.from(parts[0], 'hex');
  const encryptedText = parts[1];
  const decipher = crypto.createDecipheriv('aes-256-cbc', Buffer.from(ENCRYPTION_KEY), iv);
  let decrypted = decipher.update(encryptedText, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}

// Usage
const encryptedAadhaar = encrypt(aadhaarNumber);
await supabase.table('patta_holders').insert({ aadhaar_number: encryptedAadhaar });
```

### In Transit
- **TLS 1.3** for all API communication
- **Certificate Pinning** for mobile apps
- **VPN** for internal admin access (optional)

## 8.3 Audit Trail

Every data modification is logged automatically via trigger:

```sql
SELECT
    al.created_at,
    al.table_name,
    al.action,
    al.user_id,
    up.full_name,
    up.role,
    al.old_values ->> 'status' as old_status,
    al.new_values ->> 'status' as new_status
FROM audit_logs al
JOIN user_profiles up ON al.user_id = up.id
WHERE al.table_name = 'fra_claims'
AND al.record_id = 'specific-claim-uuid'
ORDER BY al.created_at DESC;
```

**Audit Retention**: 7 years (government compliance)

## 8.4 Disaster Recovery

### Backup Strategy

1. **Database Backups**
   - Automated daily full backups (Supabase managed)
   - Point-in-time recovery (PITR) enabled
   - Retention: 30 days
   - Geo-redundant storage

2. **Document Backups**
   - Object storage replication across regions
   - Versioning enabled
   - Retention: Indefinite (legal requirement)

3. **Recovery Objectives**
   - RPO (Recovery Point Objective): <1 hour
   - RTO (Recovery Time Objective): <4 hours

### DR Plan

```
Disaster Scenario: Database Failure
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Detection: Automated monitoring alerts (< 5 min)
2. Assessment: DR team evaluates extent (< 15 min)
3. Failover: Switch to replica database (< 30 min)
4. Verification: Run integrity checks (< 1 hour)
5. Communication: Notify users via status page
6. Root Cause Analysis: Post-incident review (< 24 hours)
```

## 8.5 Data Integrity Validation

### Automated Checks

```sql
-- Daily data integrity check
CREATE OR REPLACE FUNCTION daily_integrity_check()
RETURNS TABLE(check_name TEXT, status TEXT, details TEXT) AS $$
BEGIN
    -- Check 1: Claims without village
    RETURN QUERY
    SELECT
        'Claims without village' as check_name,
        CASE WHEN COUNT(*) = 0 THEN 'PASS' ELSE 'FAIL' END as status,
        COUNT(*)::TEXT || ' claims found' as details
    FROM fra_claims WHERE village_id IS NULL;

    -- Check 2: Spatial data without geometry
    RETURN QUERY
    SELECT
        'Spatial data without geometry' as check_name,
        CASE WHEN COUNT(*) = 0 THEN 'PASS' ELSE 'FAIL' END as status,
        COUNT(*)::TEXT || ' records found' as details
    FROM claim_spatial_data WHERE geometry IS NULL;

    -- Check 3: Duplicate claim numbers
    RETURN QUERY
    SELECT
        'Duplicate claim numbers' as check_name,
        CASE WHEN COUNT(*) = 0 THEN 'PASS' ELSE 'FAIL' END as status,
        COUNT(*)::TEXT || ' duplicates found' as details
    FROM (
        SELECT claim_number, COUNT(*) as cnt
        FROM fra_claims
        GROUP BY claim_number
        HAVING COUNT(*) > 1
    ) dups;

END;
$$ LANGUAGE plpgsql;

-- Schedule daily at 2 AM
-- (via pg_cron extension or external scheduler)
```

---

# SECTION 9: IMPLEMENTATION ROADMAP

## 9.1 MVP Definition

**Minimum Viable Product (3 months):**

✅ Core Features:
1. Document upload and OCR+NER integration (existing module)
2. Basic claim record management (CRUD)
3. Village and district hierarchy
4. Simple map viewer with FRA claims (points only)
5. Role-based dashboards (3 roles: National, State, District)
6. Manual data entry interface
7. Basic reports (district statistics)

❌ NOT in MVP:
- Advanced spatial analysis
- DSS recommendations
- CSS scheme integration
- Asset mapping
- Dispute resolution workflow
- Mobile app

## 9.2 6-Month Implementation Roadmap

### Month 1-2: Foundation
**Team: 2 Backend, 1 Frontend, 1 DevOps**

Week 1-2:
- [ ] Set up development environment
- [ ] Initialize Supabase project
- [ ] Create database schema (core tables)
- [ ] Set up Git repository and CI/CD
- [ ] Configure authentication (Supabase Auth)

Week 3-4:
- [ ] Integrate existing OCR+NER module
- [ ] Build document upload API
- [ ] Create job queue (Redis Streams)
- [ ] Implement basic validation pipeline

Week 5-6:
- [ ] Build admin data entry interface
- [ ] Implement villages/districts management
- [ ] Create user management (roles)

Week 7-8:
- [ ] Develop claim management API (CRUD)
- [ ] Build district officer dashboard (basic)
- [ ] Implement approval workflow

### Month 3-4: WebGIS & Visualization
**Team: +1 GIS Developer, +1 UI Designer**

Week 9-10:
- [ ] Set up PostGIS and spatial tables
- [ ] Implement shapefile upload
- [ ] Create vector tile server
- [ ] Build basic map viewer (MapLibre)

Week 11-12:
- [ ] Develop layer control panel
- [ ] Implement spatial queries (overlap detection)
- [ ] Create village profile page
- [ ] Add map popups and interactions

Week 13-14:
- [ ] Build state admin dashboard
- [ ] Create district statistics view
- [ ] Implement chart visualizations

Week 15-16:
- [ ] Design and implement UI theme
- [ ] Add multilingual support (Hindi + English)
- [ ] User testing and refinement

### Month 5: DSS & Integration
**Team: +1 Data Scientist**

Week 17-18:
- [ ] Design CSS scheme database
- [ ] Implement eligibility rule engine
- [ ] Create scheme matching logic
- [ ] Build recommendation API

Week 19-20:
- [ ] Develop DSS dashboard UI
- [ ] Implement scoring algorithms
- [ ] Create gap analysis views
- [ ] Add scheme enrollment tracking

### Month 6: Testing & Deployment
**Team: +1 QA Engineer**

Week 21-22:
- [ ] Comprehensive testing (unit, integration, E2E)
- [ ] Security audit and penetration testing
- [ ] Performance optimization
- [ ] Load testing (10K concurrent users)

Week 23:
- [ ] User acceptance testing (UAT) with pilot district
- [ ] Bug fixes and refinements
- [ ] Documentation (user manuals, API docs)

Week 24:
- [ ] Production deployment
- [ ] Training for state/district officers
- [ ] Monitoring setup
- [ ] Go-live!

## 9.3 Team Structure

### Core Team (Month 1-6)

**Technical Lead (1)**
- Overall architecture
- Code review
- Technology decisions

**Backend Developers (2)**
- API development
- Database design
- Integration services
- Queue management

**Frontend Developers (2)**
- React application
- Dashboard UIs
- Map components
- Responsive design

**GIS Developer (1)**
- PostGIS setup
- Spatial queries
- Tile server
- Shapefile processing

**Data Scientist (1)**
- DSS algorithms
- Scoring models
- Analytics

**UI/UX Designer (1)**
- Interface design
- User flows
- Government aesthetics

**DevOps Engineer (1)**
- Infrastructure setup
- CI/CD pipelines
- Monitoring
- Deployment

**QA Engineer (1)**
- Test planning
- Automated testing
- UAT coordination

**Project Manager (1)**
- Sprint planning
- Stakeholder communication
- Timeline management

**Total: 11 people**

### Post-Launch Team (Ongoing)

- 1 Tech Lead
- 2 Backend Developers
- 1 Frontend Developer
- 1 GIS Developer
- 1 DevOps
- Support team (3-4 people)

## 9.4 Technology Stack Summary

### Backend
- **Runtime**: Node.js 20 LTS
- **Framework**: Express.js / Fastify
- **Database**: Supabase (PostgreSQL + PostGIS)
- **Queue**: Redis Streams
- **Storage**: Supabase Storage
- **OCR Integration**: Python FastAPI wrapper

### Frontend
- **Framework**: Next.js 14 (React 18)
- **Language**: TypeScript
- **UI Library**: Shadcn/ui + Tailwind CSS
- **Map**: MapLibre GL JS
- **State**: Zustand
- **Data Fetching**: React Query

### GIS
- **Database**: PostGIS
- **Processing**: GDAL, GeoPandas
- **Tile Server**: Martin / pg_tileserv
- **Formats**: GeoJSON, Shapefiles, KML

### Infrastructure
- **Hosting**: Supabase Cloud (or NIC Cloud)
- **CDN**: Cloudflare / AWS CloudFront
- **Monitoring**: Grafana + Prometheus
- **Logging**: Loki or ELK Stack

## 9.5 Hosting Recommendation

### Option 1: Supabase Cloud (Recommended for MVP)
**Pros:**
- Fastest time to market
- Managed PostgreSQL + PostGIS
- Built-in auth, storage, real-time
- Auto-scaling
- Good for <100K users

**Cons:**
- Vendor lock-in
- Costs scale with usage
- Data sovereignty concerns

**Cost**: ~$25-100/month (dev), ~$500-2000/month (production)

### Option 2: NIC Cloud (Government Requirement)
**Pros:**
- Government-approved
- Data sovereignty
- Compliance-friendly
- Lower cost (subsidized)

**Cons:**
- Slower provisioning
- Manual scaling
- Less managed services
- Requires more DevOps

**Deployment**: Kubernetes cluster on NIC Cloud, self-hosted Supabase or PostgreSQL

### Option 3: Hybrid
- Development: Supabase Cloud
- Production: NIC Cloud with self-hosted stack

**Recommended for national rollout**

## 9.6 Risks & Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| OCR+NER module integration issues | High | Medium | Early integration testing, fallback manual entry |
| Spatial data quality issues | High | High | Validation pipeline, manual review queue |
| User adoption resistance | Medium | Medium | Training programs, phased rollout |
| Performance at scale | High | Medium | Load testing, caching, database optimization |
| Data security breach | Critical | Low | Security audit, encryption, regular pen-testing |
| Vendor lock-in (Supabase) | Medium | Low | Abstract database layer, plan migration path |
| Scope creep | Medium | High | Strict MVP definition, change control process |
| Insufficient GIS expertise | High | Medium | Hire experienced GIS developer, external consultant |

## 9.7 Success Metrics

**Technical KPIs:**
- API response time: <500ms (p95)
- Map load time: <3 seconds
- Document processing: <5 minutes per doc
- System uptime: >99.5%
- Database query time: <100ms (p95)

**Business KPIs:**
- Claims digitized: 10,000 in Month 1
- User adoption: 80% of district officers active
- Data accuracy: >95% (post-validation)
- Dispute resolution time: <30 days
- Scheme enrollment increase: +20%

---

# APPENDICES

## Appendix A: API Endpoints Reference

```
# Document Management
POST   /api/v1/documents/upload
GET    /api/v1/documents/{id}/status
POST   /api/v1/documents/{id}/review
PUT    /api/v1/documents/{id}/correct

# Claims
GET    /api/v1/claims
POST   /api/v1/claims
GET    /api/v1/claims/{id}
PUT    /api/v1/claims/{id}
DELETE /api/v1/claims/{id}
GET    /api/v1/claims/{id}/spatial
POST   /api/v1/claims/bulk-approve

# Spatial
POST   /api/v1/spatial/shapefile/upload
GET    /api/v1/spatial/layers
GET    /api/v1/spatial/tiles/{z}/{x}/{y}
POST   /api/v1/spatial/query/intersect

# DSS
GET    /api/v1/dss/recommendations/{beneficiary_id}
GET    /api/v1/dss/village-gap/{village_id}
POST   /api/v1/dss/simulate

# Analytics
GET    /api/v1/analytics/district/{id}/stats
GET    /api/v1/analytics/state/{code}/summary
GET    /api/v1/analytics/reports/generate
```

## Appendix B: Glossary

- **FRA**: Forest Rights Act, 2006
- **IFR**: Individual Forest Rights
- **CFR**: Community Forest Rights
- **CR**: Community Rights
- **CSS**: Centrally Sponsored Schemes
- **SDLC**: Sub-Divisional Level Committee
- **DLC**: District Level Committee
- **NTFP**: Non-Timber Forest Produce
- **MoTA**: Ministry of Tribal Affairs

## Appendix C: Sample Data

Sample claim JSON: (Already shown in Section 3.2)
Sample village profile: (Already shown in Section 5.4)

---

# CONCLUSION

This technical blueprint provides a complete, implementation-ready architecture for the FRA Atlas and WebGIS DSS. The system is designed to:

✅ Integrate existing OCR+NER modules without redesign
✅ Support future AI asset mapping via pluggable architecture
✅ Scale from 4 states to national rollout
✅ Meet government security and compliance requirements
✅ Deliver premium user experience to officials
✅ Enable data-driven policy decisions through DSS

**Next Steps:**
1. Secure budget approval and team hiring
2. Set up development environment (Week 1)
3. Begin Month 1 sprint planning
4. Engage with pilot district for UAT planning
5. Start integration testing with existing OCR+NER module

**Timeline:** 6 months to production-ready MVP
**Budget:** ₹40-60 lakhs (team + infrastructure)
**Impact:** 1M+ FRA beneficiaries empowered

---

*Document prepared by: Senior GovTech Systems Architect*
*Date: February 2026*
*Version: 1.0*
