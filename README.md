# FRA Atlas & WebGIS DSS

**AI-Powered Forest Rights Act Implementation Monitoring System**

[![Status](https://img.shields.io/badge/Status-Planning-yellow)]()
[![Version](https://img.shields.io/badge/Version-1.0-blue)]()

## Overview

FRA Atlas is a cloud-native, microservices-based platform for national-scale Forest Rights Act (FRA) monitoring in India. The system integrates OCR+NER for document digitization, spatial GIS analysis, and a Decision Support System (DSS) for scheme recommendations.

### Target States
- 🌳 Madhya Pradesh (MP)
- 🌳 Odisha (OD)
- 🌳 Tripura (TR)
- 🌳 Telangana (TG)

## Key Features

### Document Digitization
- Automated OCR+NER for FRA documents (PDF/PNG/JPG)
- Human-in-the-loop validation pipeline
- Structured data extraction with confidence scoring

### WebGIS Portal
- Interactive map viewer with MapLibre GL JS
- Multi-layer spatial analysis (villages, districts, forest boundaries)
- Boundary overlap detection and dispute tagging
- Village and district profile dashboards

### Decision Support System (DSS)
- CSS scheme eligibility calculation
- AI-powered recommendations for beneficiaries
- Gap analysis for scheme coverage
- Policy simulation tools

### Security & Governance
- State-wise data isolation with Row-Level Security (RLS)
- JWT-based authentication with role-based access control
- Comprehensive audit trail
- Government-grade encryption (TLS 1.3, AES-256)

## Technology Stack

### Backend
- **Runtime:** Node.js 20 LTS
- **Frameworks:** Express.js, Python FastAPI
- **Database:** Supabase (PostgreSQL + PostGIS)
- **Queue:** Redis Streams
- **Storage:** Supabase Storage

### Frontend
- **Framework:** Next.js 14 (React 18)
- **Language:** TypeScript
- **UI Library:** Shadcn/ui + Tailwind CSS
- **Map Library:** MapLibre GL JS
- **State Management:** Zustand
- **Data Fetching:** React Query

### GIS Stack
- **Database:** PostGIS
- **Processing:** GDAL, GeoPandas, Shapely
- **Tile Server:** Martin / pg_tileserv
- **Formats:** GeoJSON, Shapefiles, KML

### Infrastructure
- **Containerization:** Docker
- **Orchestration:** Kubernetes
- **CI/CD:** GitHub Actions
- **Monitoring:** Grafana + Prometheus
- **Logging:** ELK Stack / Loki

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY (Kong/Nginx)                      │
│                Authentication & Rate Limiting                     │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼────────┐  ┌────────▼────────┐  ┌────────▼─────────┐
│   Document     │  │   WebGIS        │  │   DSS Engine     │
│   Ingestion    │  │   Portal        │  │   Service        │
└───────┬────────┘  └─────────────────┘  └────────┬─────────┘
        │                                           │
┌───────▼───────────────────────────────────────────▼─────────┐
│           MESSAGE QUEUE (RabbitMQ / Redis Streams)          │
└─────────────────────────────────────────────────────────────┘
        │                  │                  │
┌───────▼────────┐  ┌──────▼─────────┐  ┌───▼──────────────┐
│  OCR+NER       │  │  GIS Processing│  │  Asset Mapping   │
│  Integration   │  │  Engine        │  │  Service (Future)│
└───────┬────────┘  └──────┬─────────┘  └───┬──────────────┘
        │                  │                 │
        └──────────────────┼─────────────────┘
                           │
┌──────────────────────────▼──────────────────────────┐
│            SUPABASE DATABASE LAYER                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ FRA DB   │  │ Spatial  │  │Analytics │          │
│  │          │  │ (PostGIS)│  │Warehouse │          │
│  └──────────┘  └──────────┘  └──────────┘          │
└──────────────────────────────────────────────────────┘
```

## Project Status

**Current Phase:** Planning & Documentation ✅

- [x] Technical Blueprint Complete
- [x] Architecture Design
- [x] Database Schema
- [x] API Specification
- [x] Security Design
- [x] Implementation Roadmap
- [ ] Development Environment Setup
- [ ] Team Assembly
- [ ] Implementation (6-month timeline)

## Documentation

- 📋 **[Technical Blueprint](TECHNICAL_BLUEPRINT.md)** - Comprehensive 3,678-line technical specification
- 📊 **[Repository Review](REPOSITORY_REVIEW.md)** - Complete project analysis and assessment
- 📚 **API Documentation** - (Coming soon)
- 🔒 **Security Documentation** - (Coming soon)

## Implementation Roadmap

### MVP (3 months)
- ✅ Document upload and OCR+NER integration
- ✅ Basic claim record management (CRUD)
- ✅ Village and district hierarchy
- ✅ Simple map viewer with FRA claims
- ✅ Role-based dashboards (3 roles)
- ✅ Manual data entry interface
- ✅ Basic reports (district statistics)

### Full System (6 months)
- **Month 1-2:** Foundation (Auth, Database, Document Upload)
- **Month 3-4:** WebGIS & Visualization (Maps, Spatial Queries)
- **Month 5:** DSS & Integration (Scheme Recommendations)
- **Month 6:** Testing & Deployment (UAT, Go-live)

## Team Structure

**Required Team (11 people):**
- 1 Technical Lead
- 2 Backend Developers
- 2 Frontend Developers
- 1 GIS Developer
- 1 Data Scientist
- 1 UI/UX Designer
- 1 DevOps Engineer
- 1 QA Engineer
- 1 Project Manager

## Getting Started

### Prerequisites
- Node.js 20 LTS
- Python 3.11+
- Docker & Docker Compose
- PostgreSQL 15+ with PostGIS
- Redis

### Installation

```bash
# Clone the repository
git clone https://github.com/2024cs0097-stack/fra.git
cd fra

# Install dependencies (when implemented)
npm install

# Set up environment variables
cp .env.example .env

# Start development environment
docker-compose up -d
```

**Note:** Implementation is not yet started. See [REPOSITORY_REVIEW.md](REPOSITORY_REVIEW.md) for next steps.

## Success Metrics

### Technical KPIs
- API response time: <500ms (p95)
- Map load time: <3 seconds
- Document processing: <5 minutes per doc
- System uptime: >99.5%
- Database query time: <100ms (p95)

### Business KPIs
- Claims digitized: 10,000 in Month 1
- User adoption: 80% of district officers active
- Data accuracy: >95% (post-validation)
- Dispute resolution time: <30 days
- Scheme enrollment increase: +20%

## Security

- 🔐 JWT-based authentication with Supabase Auth
- 🛡️ Row-level security (RLS) for state-wise data isolation
- 🔒 TLS 1.3 for data in transit
- 💾 AES-256 encryption for data at rest
- 📝 Comprehensive audit trail
- 👥 Role-based access control (7 roles)

## Contributing

This project is in the planning phase. Contributions will be welcome once development begins.

## License

(To be determined)

## Contact

- **Ministry of Tribal Affairs (MoTA)**
- **Project Team:** (To be assembled)

## Acknowledgments

- Existing OCR+NER module providers
- Open-source GIS community
- Government of India - Digital India Initiative

---

**🌳 Empowering Forest Rights through Technology 🌳**
