# PharmChecker 💊

A pharmacy license verification system that imports search results from state boards, automatically scores address matches, and provides a web interface for manual review and validation.

## Overview

PharmChecker streamlines pharmacy license verification across multiple U.S. states by:

- **Importing** pharmacy data (CSV) and state board search results (JSON) with screenshots
- **Deduplicating** images using SHA256 content hashing for efficient storage
- **Scoring** addresses automatically using fuzzy matching algorithms  
- **Reviewing** results through an interactive Streamlit web dashboard
- **Validating** findings manually with audit trail support
- **Exporting** verified results for compliance reporting

**Key Features:** Supabase cloud database, dataset versioning, lazy scoring computation, multi-user sessions, natural key linking (no hardcoded IDs), comprehensive validation system with overrides.

## Architecture Overview

```
CSV/JSON Files → Import System → Supabase Database → Scoring Engine → Web Interface
                       ↓                   ↓
                 SHA256 Image Storage   Versioned Datasets + Screenshots + Validations
```

- **Supabase cloud database** with REST API endpoints for all operations
- **SHA256 image deduplication** prevents storage of duplicate screenshots
- **Lazy scoring system** computes address matches only when needed
- **Multi-user support** with session management and authentication  
- **Natural key linking** using pharmacy names/license numbers (not internal IDs)

## Quick Start

### Prerequisites
- **Database**: Supabase account and project
- **Runtime**: Python 3.8+
- **Storage**: 2GB free disk space for data and screenshots
- **Authentication** (optional): GitHub OAuth app for production

### Installation

```bash
# 1. Clone repository
git clone [repository-url]
cd pharmchecker

# 2. Create Python virtual environment and install dependencies
make venv
source venv/bin/activate  # You'll need to run this in each new terminal

# 3. Configure Supabase
cp .env.example .env
# Edit with your Supabase project credentials:
# - SUPABASE_URL
# - SUPABASE_ANON_KEY
# - SUPABASE_SERVICE_KEY

# 4. Set up authentication mode
# For local development:
echo "AUTH_MODE=local" >> .env

# For production with GitHub OAuth:
# Follow docs/GITHUB_AUTH_SETUP.md to configure GitHub OAuth
# Then set: AUTH_MODE=github

# 5. Set up Supabase schema
# Run this SQL in your Supabase Dashboard SQL Editor:
# Copy/paste: database/migrations/supabase_setup_consolidated.sql

# 6. Initialize and verify database
make setup                                      # Verify Supabase connection
python3 database/migrations/migrate.py --verify # Verify schema setup

# 7. Run system test to verify installation
python3 -m pytest tests/integration/test_system.py
```

#### About Virtual Environments

Modern Python (especially on macOS/Linux) requires using virtual environments to avoid system conflicts. The `make venv` command creates an isolated Python environment for this project.

**Important**: After opening a new terminal, always activate the virtual environment:
```bash
cd pharmchecker
source venv/bin/activate  # You'll see (venv) in your prompt
```

### Launch Application

```bash
# Make sure virtual environment is activated
source venv/bin/activate  # If not already activated

# Import sample data and verify setup
make dev  # Imports test pharmacies and state search results

# Start web interface
streamlit run frontend/app.py
# Opens at http://localhost:8501
```

The web dashboard includes:
- **Dataset Selection** - Load pharmacy, state search, and validation datasets
- **Results Matrix** - Review all pharmacy-state combinations with scores
- **Detail Views** - Examine individual search results and screenshots  
- **Validation Tools** - Mark results as verified with reasons
- **Export Functions** - Download results as CSV for reporting

## Authentication System

PharmChecker supports two authentication modes:

### Local Development Mode
- Set `AUTH_MODE=local` in your `.env` file
- Creates an automatic admin user for development
- No external authentication required
- Ideal for testing and development

### GitHub OAuth (Production)
- Set `AUTH_MODE=github` in your `.env` file
- Integrates with GitHub OAuth for secure production authentication
- Users authenticate with their GitHub accounts
- New users automatically created in the system
- See `docs/GITHUB_AUTH_SETUP.md` for full configuration guide

### User Roles
- **admin**: Full access to all features and admin operations
- **user**: Standard access to view and validate results (default for new users)

### Environment Variables

Required for both modes:
```bash
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_KEY=your-service-key
AUTH_MODE=local  # or 'github'
```

Optional for local mode:
```bash
DEFAULT_USER_EMAIL=admin@localhost
DEFAULT_USER_ROLE=admin
```

## Database Migration System

PharmChecker uses Supabase for cloud database operations with manual schema setup via SQL Dashboard.

### Migration Components

```
migrations/
├── migrate.py                        # Universal migration runner
├── migrations/                       # Versioned migration files
│   ├── 20240101000000_initial_schema.sql
│   ├── 20240101000001_comprehensive_functions.sql
│   └── 20240101000002_indexes_and_performance.sql
├── supabase_setup_consolidated.sql   # Complete Supabase setup (all-in-one)
├── config.toml                       # Migration configuration
└── README.md                         # Migration documentation
```

### Migration Commands

```bash
# Check migration status
python3 database/migrations/migrate.py --status        # Show schema documentation
python3 database/migrations/migrate.py --verify        # Verify Supabase setup

# Schema setup for Supabase
# Manual setup required via Supabase Dashboard SQL Editor
# Use: database/migrations/supabase_setup_consolidated.sql

# Setup with migrations (recommended)
python3 database/setup.py                                 # Auto-detects and applies migrations
```

### Required Environment Variables

Create a `.env` file in the project root with the following Supabase configuration:

```bash
# Required - Get from your Supabase project dashboard
SUPABASE_URL=https://your-project-id.supabase.co
SUPABASE_ANON_KEY=your-anon-key-here
SUPABASE_SERVICE_KEY=your-service-role-key-here

# Optional - Application settings
API_CACHE_TTL=300
API_RETRY_COUNT=3
LOGGING_LEVEL=INFO
```

**Where to find these values:**
- Log into your [Supabase dashboard](https://supabase.com/dashboard)
- Go to Settings → API
- Copy the URL and both API keys

### Migration Features

- ✅ **Version Tracking**: All migrations tracked in `pharmchecker_migrations` table
- ✅ **Idempotent**: Safe to run multiple times, skips applied migrations
- ✅ **Schema Consistency**: All tables, functions, and indexes in Supabase
- ✅ **Atomic Operations**: Each migration runs in a transaction
- ✅ **Rollback Ready**: Migration tracking supports future rollback features

## Project Structure

```
pharmchecker/
├── README.md                   # This file
├── requirements.txt            # Python dependencies
├── frontend/
│   └── app.py                 # Streamlit web interface
├── database/
│   ├── setup.py               # Database initialization
│   ├── schema.sql             # Legacy database schema
│   └── migrations/            # Unified migration system
│       ├── migrate.py         # Migration documentation and verification tool
│       └── supabase_setup_consolidated.sql  # Complete Supabase setup
├── tests/
│   └── integration/
│       └── test_system.py     # End-to-end validation
├── importers/                 # Data import system
│   ├── resilient_importer.py  # High-performance production importer
│   ├── api_importer.py        # API-based importer (Supabase)
│   ├── pharmacies.py          # Legacy CSV pharmacy importer
│   ├── states.py              # Legacy JSON search results importer
│   ├── scoring.py             # Address matching engine
│   └── validated.py           # Manual validation importer
├── api_poc/                   # API and GUI proof of concept
│   ├── gui/                   # Streamlit GUI for API testing
│   └── postgrest/             # PostgREST API server
├── frontend/utils/            # GUI utilities
│   ├── display.py             # UI components  
│   ├── image_storage.py       # SHA256 image storage system
│   ├── api_database.py        # API database management
│   └── session.py             # Session management
├── docs/                      # Comprehensive documentation
└── data/                      # Sample data for testing
```

## Data Import System

PharmChecker includes a **high-performance Resilient Importer** for production-scale data processing of state board search results with screenshots.

### 🚀 Resilient Importer (Production)

For large-scale imports (500+ files), use the resilient importer:

```bash
# Import state search results with images
make import_scrape_states                    # Import with images (Supabase)

# Direct usage with options
python3 imports/resilient_importer.py \
    --states-dir "/path/to/data" \
    --tag "Aug-04-scrape" \
    --batch-size 25 \
    --max-workers 16 \
    --debug-log
```

**Features:**
- ⚡ **60x faster** than sequential processing (49s vs 23+ minutes)
- 🔄 **Resume capability** for interrupted imports  
- 📊 **Real-time progress** tracking with work state persistence
- 🛡️ **Robust error handling** with automatic retries and conflict resolution
- 🖼️ **Smart image handling** with SHA256 deduplication and cross-directory PNG resolution
- ✅ **100% reliability** with comprehensive data validation and cleaning

**Performance:** Processes 515 files with 514 images in 49 seconds with 0 failures.

See [docs/RESILIENT_IMPORTER.md](docs/RESILIENT_IMPORTER.md) for comprehensive documentation.

### Legacy Import System

For smaller datasets, use the legacy API-based importers:

```bash
# Pharmacy data (CSV)
make import_pharmacies                       # Supabase

# State search results (JSON) - small datasets only
make import_test_states                      # Supabase
```

## Typical Workflow

1. **Setup environment** (first time): `make venv && source venv/bin/activate`
2. **Import pharmacy data**: `make import_pharmacies`
3. **Import state searches**:
   - **Large datasets (500+ files)**: `make import_scrape_states` (recommended)
   - **Specific states only**: `python3 importers/resilient_importer.py --states-dir /path/to/data --states FL TX CA --tag my_tag`
   - **Small test datasets**: `make import_test_states`
4. **Launch web interface**: `streamlit run frontend/app.py`
5. **Select datasets**: Choose pharmacy and state datasets in sidebar
6. **Review results**: System auto-computes address match scores (85+ = match)
7. **Validate findings**: Mark verified results with reasons
8. **Export report**: Download CSV with all results and validations

## Complete Data Update Workflow (Production)

To collect fresh data from state websites and update your database:

### Step 1: Scrape State Websites
```bash
# Activate virtual environment
source venv/bin/activate

# Scrape ALL available states (currently FL, CA, NY, PA, TX, MI, MA, OH, IL, NJ, VA)
python scraper/01_download.py --input data/pharmacies.csv

# Or scrape specific states only
python scraper/01_download.py --state FL,TX,CA --input data/pharmacies.csv

# Or scrape a single pharmacy across all states
python scraper/01_download.py --name "Belmar Pharmacy"

# Output: Creates HTML/PNG files in scrape_data/YYYY-MM-DD/STATE/
```

### Step 2: Parse HTML to JSON
```bash
# Parse today's scraped data (auto-detects today's date)
python parser/02_parse.py

# Or parse specific date
python parser/02_parse.py --dir scrape_data/2025-01-14

# Output: Creates *_parse.json files alongside HTML files
```

### Step 3: Import to Database
```bash
# Import parsed JSON files using Resilient Importer
python importers/resilient_importer.py \
    --states-dir scrape_data/$(date +%F) \
    --tag production_$(date +%F)

# Or use the Makefile shortcut
make import_scrape_states STATES_DIR=scrape_data/$(date +%F) TAG=production_$(date +%F)

# Output: Data imported to Supabase with dataset tag
```

### Step 4: View and Validate
```bash
# Launch web interface
streamlit run frontend/app.py

# In the web UI:
# 1. Select your new dataset (e.g., "production_2025-01-14")
# 2. Review matches and scores
# 3. Validate results as needed
```

### Complete Example (Daily Update)
```bash
# Full pipeline for daily pharmacy license verification across all states
source venv/bin/activate
python scraper/01_download.py --input data/pharmacies.csv  # Scrapes all 11 states
python parser/02_parse.py
python importers/resilient_importer.py --states-dir scrape_data/$(date +%F) --tag daily_$(date +%F)
streamlit run frontend/app.py
```

**Note**: Scraping all states can take 30-60 minutes depending on the number of pharmacies. The scraper includes a daily cache, so running it again the same day will skip already-scraped pharmacies.

### API POC Workflow (Advanced)

1. **Start PostgREST API**: `cd api_poc/postgrest && ./postgrest postgrest.conf`
2. **Launch Admin GUI**: `streamlit run frontend/app.py --server.port 8502`
3. **Access Supabase backend**: All operations via Supabase REST API
4. **Import via API**: Use Data Manager page for file uploads and transfers
5. **Comprehensive results**: Query Supabase through unified interface

## Key Features

### 🔄 Transparent Client-Side Scoring System
- Address matching computed **transparently** when dataset combinations are first accessed
- **Client-side execution** of `scoring_plugin.py` for cloud compatibility
- **API-first architecture** - all scoring via REST endpoints, no database functions
- Intelligent fuzzy matching algorithm with street/city/state/ZIP components (70% street, 30% location)
- Results cached permanently in database for performance
- **One-time computation** per dataset pair - never recomputed unless explicitly cleared

### 📊 Dataset Versioning  
- Import multiple versions of pharmacy, state, and validation data
- Mix and match any combination (e.g., "old pharmacies" + "new searches")
- No global "active" state - users work with specific dataset combinations

### 🌐 Web Interface
- **Streamlit-based** dashboard with real-time updates
- **Interactive filtering** by state, status, score ranges
- **Detail views** with side-by-side address comparisons
- **Screenshot integration** with SHA256 deduplication and automatic display
- **Export functionality** with CSV downloads

### ✅ Validation System
- **Manual overrides** for automated scoring results
- **Two validation types**: "present" (license exists) and "empty" (no license)
- **Audit trail** with reasons, timestamps, user tracking
- **Warning system** detects data changes since validation

## Key Commands

```bash
# Setup (first time only)
make venv            # Create virtual environment
source venv/bin/activate  # Activate it (needed each terminal session)

# Database management
make setup           # Initialize Supabase connection
make status          # Show current data
make clean_all       # Full reset

# Schema management
python3 database/migrations/migrate.py --status    # Show schema documentation
python3 database/migrations/migrate.py --verify    # Verify Supabase schema

# Development workflow
make dev            # Import all test data
python3 -m pytest tests/integration/test_system.py    # Run end-to-end test

# Production data import (Resilient Importer - recommended)
make import_scrape_states                 # Import state searches with images (Supabase)

# Legacy data import (API-based - for small datasets)
make import_pharmacies                    # Import pharmacy data (Supabase)
make import_test_states                   # Import state search results (Supabase)

# Direct API import (advanced usage)
python3 -m importers.api_importer pharmacies <csv_file> <tag>
python3 -m importers.api_importer states <states_dir> <tag>

# Testing
python3 test_scoring.py   # Test address matching algorithm
python3 test_gui.py       # Test web interface components
```

## Documentation

| Document | Description | For |
|----------|-------------|-----|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | System design, database schema, data flow | Developers |
| [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md) | Setup, debugging, contribution guide | Developers |
| [docs/RESILIENT_IMPORTER.md](docs/RESILIENT_IMPORTER.md) | High-performance production importer | Developers |
| [docs/USER_GUIDE.md](docs/USER_GUIDE.md) | Web interface usage, features | End Users |
| [docs/API_REFERENCE.md](docs/API_REFERENCE.md) | Functions, modules, data formats | Integrators |
| [docs/TESTING.md](docs/TESTING.md) | Test procedures, validation | QA Teams |

## System Requirements

### Minimum Requirements
- **Database**: Supabase account and project
- **Runtime**: Python 3.8+ with pip
- **Storage**: 2GB available disk space
- **Memory**: 4GB RAM

### Recommended for Production
- **Database**: Supabase Pro plan
- **Runtime**: Python 3.10+
- **Storage**: 10GB+ disk space for screenshots
- **Memory**: 8GB+ RAM for large datasets
- **Platform**: Linux or macOS (Windows via WSL2)

## Technology Stack

- **Database**: Supabase (cloud PostgreSQL with REST API)
- **Backend**: Python 3.8+, Supabase Python client
- **API Layer**: Supabase REST API (cloud-native)
- **Scoring Engine**: Client-side `scoring_plugin.py` with RapidFuzz address matching
- **Web Framework**: Streamlit 1.28+ with Plotly charts
- **Data Processing**: pandas, RapidFuzz (address matching)
- **Import System**: REST API-based importers for Supabase
- **Dependencies**: python-dotenv, python-slugify, requests
- **Migration System**: Supabase schema management with SQL migrations
- **Testing**: Built-in test suite (system_test.py)

## Project Status

✅ **Production Ready** - All core features implemented and tested:
- Database schema with versioned datasets
- Import system for pharmacies and state searches  
- Address scoring engine with 96.5% accuracy for exact matches
- Web interface with full CRUD operations
- Validation system with audit trails
- Export functionality for reporting

## Support

For issues or questions:
1. Check [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) for common issues
2. Review the system test output: `python3 system_test.py`
3. Enable debug mode in `.env`: `LOGGING_LEVEL=DEBUG`
