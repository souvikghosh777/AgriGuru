# AgriGuru Database Schema Design

## 🗄️ Database Architecture Overview

The AgriGuru platform uses a **multi-database architecture** to optimize performance, scalability, and data management:

1. **PostgreSQL** - Primary relational database for structured data
2. **ClickHouse** - Analytics database for time-series and aggregated data
3. **Redis** - Caching layer and session management
4. **Elasticsearch** - Full-text search and indexing

## 📊 PostgreSQL Schema (Primary Database)

### **Database Configuration**
```sql
-- Database creation with UTF-8 support for multilingual content
CREATE DATABASE agriguru_main 
    WITH ENCODING 'UTF8' 
    LC_COLLATE = 'en_US.UTF-8' 
    LC_CTYPE = 'en_US.UTF-8'
    TEMPLATE template0;

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";        -- For geospatial data
CREATE EXTENSION IF NOT EXISTS "pg_trgm";        -- For text search
CREATE EXTENSION IF NOT EXISTS "btree_gin";      -- For composite indexes
```

### **1. User Management Schema**

```sql
-- User Types Enum
CREATE TYPE user_type_enum AS ENUM (
    'farmer', 'expert', 'supplier', 'buyer', 
    'government', 'admin'
);

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    user_type user_type_enum NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    is_verified BOOLEAN DEFAULT FALSE,
    phone VARCHAR(20),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP WITH TIME ZONE,
    
    -- Indexes
    CONSTRAINT valid_email CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

-- User Profiles table
CREATE TABLE user_profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    profile_image_url TEXT,
    date_of_birth DATE,
    gender VARCHAR(10),
    address JSONB,  -- Flexible address structure
    location GEOMETRY(POINT, 4326),  -- Geographic coordinates
    preferred_language VARCHAR(10) DEFAULT 'en',
    bio TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Farmer-specific profiles
CREATE TABLE farmer_profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    farm_name VARCHAR(255),
    total_farm_size DECIMAL(10,2),  -- in acres
    farm_size_unit VARCHAR(20) DEFAULT 'acres',
    farming_experience_years INTEGER,
    farming_type VARCHAR(50),  -- organic, conventional, mixed
    primary_crops JSONB,  -- Array of primary crops grown
    water_sources JSONB,  -- Array of water sources
    soil_types JSONB,  -- Array of soil types in farm
    machinery_owned JSONB,  -- Array of owned machinery
    annual_income_range VARCHAR(50),
    education_level VARCHAR(50),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Expert profiles
CREATE TABLE expert_profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    specializations JSONB NOT NULL,  -- Array of specialization areas
    qualifications JSONB,  -- Array of educational qualifications
    experience_years INTEGER,
    languages_spoken JSONB,  -- Array of languages
    consultation_fee DECIMAL(10,2),
    consultation_currency VARCHAR(3) DEFAULT 'INR',
    availability_schedule JSONB,  -- Weekly availability schedule
    rating DECIMAL(3,2) DEFAULT 0.0,
    total_consultations INTEGER DEFAULT 0,
    certification_documents JSONB,  -- Array of certification file URLs
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Supplier profiles
CREATE TABLE supplier_profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    company_name VARCHAR(255) NOT NULL,
    business_type VARCHAR(100),  -- equipment, seeds, fertilizers, etc.
    gst_number VARCHAR(20),
    pan_number VARCHAR(20),
    business_address JSONB,
    service_areas JSONB,  -- Array of service locations
    product_categories JSONB,  -- Array of product categories
    certifications JSONB,  -- Business certifications
    rating DECIMAL(3,2) DEFAULT 0.0,
    total_transactions INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### **2. Agricultural Data Schema**

```sql
-- Crop master data
CREATE TABLE crops (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    scientific_name VARCHAR(255),
    local_names JSONB,  -- Names in different languages
    category VARCHAR(100),  -- cereal, pulse, oilseed, etc.
    season VARCHAR(50),  -- kharif, rabi, zaid
    growth_duration_days INTEGER,
    water_requirement VARCHAR(50),
    suitable_soil_types JSONB,
    climate_requirements JSONB,
    nutritional_requirements JSONB,
    market_demand_rating INTEGER CHECK (market_demand_rating BETWEEN 1 AND 5),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Crop varieties
CREATE TABLE crop_varieties (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    crop_id UUID REFERENCES crops(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    characteristics JSONB,
    yield_potential DECIMAL(10,2),
    disease_resistance JSONB,
    maturity_days INTEGER,
    seed_rate DECIMAL(10,2),
    recommended_regions JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Farm fields
CREATE TABLE farm_fields (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    farmer_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    area DECIMAL(10,2) NOT NULL,
    area_unit VARCHAR(20) DEFAULT 'acres',
    location GEOMETRY(POLYGON, 4326),  -- Field boundaries
    soil_type VARCHAR(100),
    soil_ph DECIMAL(3,1),
    soil_health_score INTEGER CHECK (soil_health_score BETWEEN 1 AND 100),
    irrigation_type VARCHAR(100),
    water_source VARCHAR(100),
    elevation_meters DECIMAL(8,2),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Crop planning
CREATE TABLE crop_plans (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    farmer_id UUID REFERENCES users(id) ON DELETE CASCADE,
    field_id UUID REFERENCES farm_fields(id) ON DELETE CASCADE,
    crop_id UUID REFERENCES crops(id) ON DELETE CASCADE,
    variety_id UUID REFERENCES crop_varieties(id),
    season VARCHAR(50),
    crop_year INTEGER,
    planned_area DECIMAL(10,2),
    planting_date DATE,
    expected_harvest_date DATE,
    expected_yield DECIMAL(10,2),
    yield_unit VARCHAR(20) DEFAULT 'quintals',
    investment_budget DECIMAL(12,2),
    target_market VARCHAR(100),
    notes TEXT,
    status VARCHAR(50) DEFAULT 'planned',  -- planned, planted, growing, harvested
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Crop growth tracking
CREATE TABLE crop_growth_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    crop_plan_id UUID REFERENCES crop_plans(id) ON DELETE CASCADE,
    growth_stage VARCHAR(100),  -- germination, vegetative, flowering, etc.
    record_date DATE NOT NULL,
    plant_height_cm DECIMAL(6,2),
    plant_population INTEGER,
    health_score INTEGER CHECK (health_score BETWEEN 1 AND 10),
    photos JSONB,  -- Array of photo URLs
    observations TEXT,
    issues_identified JSONB,  -- Array of identified issues
    actions_taken JSONB,  -- Array of actions taken
    recorded_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### **3. Pest & Disease Management Schema**

```sql
-- Pest master data
CREATE TABLE pests (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    scientific_name VARCHAR(255),
    local_names JSONB,
    type VARCHAR(50),  -- insect, fungus, bacteria, virus, weed
    description TEXT,
    symptoms JSONB,  -- Array of symptoms
    affected_crops JSONB,  -- Array of crop IDs
    life_cycle JSONB,  -- Life cycle information
    favorable_conditions JSONB,
    prevention_methods JSONB,
    economic_threshold JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Treatment methods
CREATE TABLE treatments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    pest_id UUID REFERENCES pests(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50),  -- chemical, biological, cultural, organic
    active_ingredients JSONB,
    dosage JSONB,  -- Dosage information
    application_method VARCHAR(100),
    timing JSONB,  -- Application timing
    safety_precautions JSONB,
    effectiveness_rating INTEGER CHECK (effectiveness_rating BETWEEN 1 AND 5),
    cost_rating INTEGER CHECK (cost_rating BETWEEN 1 AND 5),
    environmental_impact VARCHAR(50),
    withdrawal_period_days INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Pest identification records
CREATE TABLE pest_identifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    crop_plan_id UUID REFERENCES crop_plans(id),
    images JSONB,  -- Array of uploaded image URLs
    location GEOMETRY(POINT, 4326),
    identified_pest_id UUID REFERENCES pests(id),
    confidence_score DECIMAL(4,3),  -- AI confidence score
    symptoms_reported JSONB,
    severity_level VARCHAR(50),
    expert_verified BOOLEAN DEFAULT FALSE,
    expert_id UUID REFERENCES users(id),
    treatment_recommended UUID REFERENCES treatments(id),
    status VARCHAR(50) DEFAULT 'pending',  -- pending, identified, treated
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Regional pest alerts
CREATE TABLE pest_alerts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    pest_id UUID REFERENCES pests(id) ON DELETE CASCADE,
    region JSONB,  -- Geographic region information
    severity_level VARCHAR(50),  -- low, medium, high, severe
    affected_area_km2 DECIMAL(10,2),
    start_date DATE,
    end_date DATE,
    description TEXT,
    prevention_advice TEXT,
    issued_by UUID REFERENCES users(id),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### **4. Market & Pricing Schema**

```sql
-- Market centers
CREATE TABLE markets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    type VARCHAR(100),  -- mandi, wholesale, retail, online
    location GEOMETRY(POINT, 4326),
    address JSONB,
    operating_hours JSONB,
    contact_info JSONB,
    commodities_traded JSONB,  -- Array of commodity IDs
    facilities JSONB,  -- Available facilities
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Commodity price records
CREATE TABLE commodity_prices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    crop_id UUID REFERENCES crops(id) ON DELETE CASCADE,
    variety_id UUID REFERENCES crop_varieties(id),
    market_id UUID REFERENCES markets(id) ON DELETE CASCADE,
    price_date DATE NOT NULL,
    min_price DECIMAL(10,2),
    max_price DECIMAL(10,2),
    modal_price DECIMAL(10,2),
    average_price DECIMAL(10,2),
    unit VARCHAR(20) DEFAULT 'quintal',
    currency VARCHAR(3) DEFAULT 'INR',
    volume_traded DECIMAL(12,2),
    quality_grade VARCHAR(50),
    trend VARCHAR(20),  -- increasing, decreasing, stable
    source VARCHAR(100),  -- agmarknet, manual, api
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    -- Ensure unique price per crop/market/date
    UNIQUE(crop_id, market_id, price_date, quality_grade)
);

-- Price alerts
CREATE TABLE price_alerts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    crop_id UUID REFERENCES crops(id) ON DELETE CASCADE,
    market_id UUID REFERENCES markets(id),
    target_price DECIMAL(10,2),
    alert_condition VARCHAR(20),  -- above, below, equals
    is_active BOOLEAN DEFAULT TRUE,
    triggered_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Demand forecasting
CREATE TABLE demand_forecasts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    crop_id UUID REFERENCES crops(id) ON DELETE CASCADE,
    region JSONB,
    forecast_period VARCHAR(50),  -- weekly, monthly, seasonal
    predicted_demand DECIMAL(12,2),
    predicted_price DECIMAL(10,2),
    confidence_level DECIMAL(4,3),
    factors JSONB,  -- Factors affecting demand
    model_used VARCHAR(100),
    forecast_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### **5. Expert Consultation Schema**

```sql
-- Consultation sessions
CREATE TABLE consultations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    farmer_id UUID REFERENCES users(id) ON DELETE CASCADE,
    expert_id UUID REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(50),  -- video, audio, chat, field_visit
    topic VARCHAR(255),
    description TEXT,
    scheduled_at TIMESTAMP WITH TIME ZONE,
    duration_minutes INTEGER,
    status VARCHAR(50) DEFAULT 'scheduled',  -- scheduled, in_progress, completed, cancelled
    fee_amount DECIMAL(10,2),
    payment_status VARCHAR(50) DEFAULT 'pending',
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    feedback TEXT,
    session_recording_url TEXT,
    session_notes TEXT,
    follow_up_required BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- AI chat conversations
CREATE TABLE ai_conversations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    session_id VARCHAR(255),
    conversation_context JSONB,  -- Context information
    total_messages INTEGER DEFAULT 0,
    language VARCHAR(10) DEFAULT 'en',
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- AI chat messages
CREATE TABLE ai_messages (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES ai_conversations(id) ON DELETE CASCADE,
    message_type VARCHAR(20),  -- user, assistant
    content TEXT NOT NULL,
    attachments JSONB,  -- Array of attachment URLs
    confidence_score DECIMAL(4,3),
    processing_time_ms INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Knowledge base articles
CREATE TABLE knowledge_articles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title VARCHAR(500) NOT NULL,
    content TEXT NOT NULL,
    summary TEXT,
    category VARCHAR(100),
    tags JSONB,  -- Array of tags
    author_id UUID REFERENCES users(id),
    language VARCHAR(10) DEFAULT 'en',
    difficulty_level VARCHAR(20),  -- beginner, intermediate, advanced
    estimated_read_time INTEGER,  -- in minutes
    view_count INTEGER DEFAULT 0,
    like_count INTEGER DEFAULT 0,
    is_featured BOOLEAN DEFAULT FALSE,
    is_published BOOLEAN DEFAULT FALSE,
    published_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### **6. Equipment & Marketplace Schema**

```sql
-- Equipment categories
CREATE TABLE equipment_categories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    parent_category_id UUID REFERENCES equipment_categories(id),
    description TEXT,
    image_url TEXT,
    sort_order INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE
);

-- Equipment catalog
CREATE TABLE equipment (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    category_id UUID REFERENCES equipment_categories(id) ON DELETE CASCADE,
    brand VARCHAR(100),
    model VARCHAR(100),
    description TEXT,
    specifications JSONB,
    images JSONB,  -- Array of image URLs
    features JSONB,  -- Array of features
    power_rating VARCHAR(50),
    fuel_type VARCHAR(50),
    weight_kg DECIMAL(8,2),
    dimensions JSONB,  -- Length, width, height
    warranty_info JSONB,
    is_rental_available BOOLEAN DEFAULT FALSE,
    is_purchase_available BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Equipment rental listings
CREATE TABLE equipment_rentals (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    equipment_id UUID REFERENCES equipment(id) ON DELETE CASCADE,
    supplier_id UUID REFERENCES users(id) ON DELETE CASCADE,
    daily_rate DECIMAL(10,2),
    hourly_rate DECIMAL(10,2),
    monthly_rate DECIMAL(10,2),
    security_deposit DECIMAL(10,2),
    minimum_rental_period INTEGER,  -- in hours
    maximum_rental_period INTEGER,  -- in hours
    availability_schedule JSONB,
    pickup_location GEOMETRY(POINT, 4326),
    delivery_available BOOLEAN DEFAULT FALSE,
    delivery_radius_km INTEGER,
    delivery_charges DECIMAL(8,2),
    terms_conditions TEXT,
    is_available BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Equipment rental bookings
CREATE TABLE rental_bookings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    rental_id UUID REFERENCES equipment_rentals(id) ON DELETE CASCADE,
    renter_id UUID REFERENCES users(id) ON DELETE CASCADE,
    start_datetime TIMESTAMP WITH TIME ZONE NOT NULL,
    end_datetime TIMESTAMP WITH TIME ZONE NOT NULL,
    total_hours INTEGER,
    rental_amount DECIMAL(10,2),
    security_deposit DECIMAL(10,2),
    delivery_required BOOLEAN DEFAULT FALSE,
    delivery_address JSONB,
    status VARCHAR(50) DEFAULT 'pending',  -- pending, confirmed, in_use, completed, cancelled
    payment_status VARCHAR(50) DEFAULT 'pending',
    pickup_photos JSONB,
    return_photos JSONB,
    damage_report TEXT,
    additional_charges DECIMAL(10,2) DEFAULT 0,
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    review TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### **7. Financial Services Schema**

```sql
-- Loan products
CREATE TABLE loan_products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    provider VARCHAR(255),
    type VARCHAR(100),  -- crop_loan, equipment_loan, working_capital
    min_amount DECIMAL(12,2),
    max_amount DECIMAL(12,2),
    interest_rate DECIMAL(5,2),
    tenure_months INTEGER,
    eligibility_criteria JSONB,
    required_documents JSONB,
    processing_fee DECIMAL(10,2),
    collateral_required BOOLEAN DEFAULT FALSE,
    government_subsidized BOOLEAN DEFAULT FALSE,
    subsidy_rate DECIMAL(5,2),
    description TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Loan applications
CREATE TABLE loan_applications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    applicant_id UUID REFERENCES users(id) ON DELETE CASCADE,
    loan_product_id UUID REFERENCES loan_products(id) ON DELETE CASCADE,
    application_number VARCHAR(100) UNIQUE,
    requested_amount DECIMAL(12,2),
    purpose TEXT,
    tenure_months INTEGER,
    application_data JSONB,  -- Complete application form data
    documents JSONB,  -- Array of uploaded document URLs
    status VARCHAR(50) DEFAULT 'draft',  -- draft, submitted, under_review, approved, rejected, disbursed
    rejection_reason TEXT,
    approved_amount DECIMAL(12,2),
    approved_rate DECIMAL(5,2),
    disbursement_date DATE,
    emi_amount DECIMAL(10,2),
    first_emi_date DATE,
    loan_account_number VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Crop insurance
CREATE TABLE insurance_products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    provider VARCHAR(255),
    type VARCHAR(100),  -- crop_insurance, livestock_insurance, equipment_insurance
    coverage_types JSONB,  -- Array of coverage types
    premium_rate DECIMAL(5,2),
    sum_assured_max DECIMAL(12,2),
    eligibility_criteria JSONB,
    coverage_area JSONB,  -- Geographic coverage
    claim_process JSONB,
    exclusions JSONB,
    government_subsidized BOOLEAN DEFAULT FALSE,
    subsidy_rate DECIMAL(5,2),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Payment transactions
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    transaction_type VARCHAR(50),  -- payment, refund, commission
    amount DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'INR',
    payment_method VARCHAR(50),
    gateway_provider VARCHAR(50),
    gateway_transaction_id VARCHAR(255),
    reference_type VARCHAR(50),  -- consultation, rental, purchase, loan
    reference_id UUID,
    status VARCHAR(50) DEFAULT 'pending',  -- pending, completed, failed, refunded
    failure_reason TEXT,
    gateway_response JSONB,
    processed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### **8. Government Schemes Schema**

```sql
-- Government schemes
CREATE TABLE government_schemes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(500) NOT NULL,
    scheme_code VARCHAR(100) UNIQUE,
    category VARCHAR(100),  -- subsidy, loan, insurance, training
    implementing_agency VARCHAR(255),
    beneficiary_type JSONB,  -- Array of eligible user types
    description TEXT,
    objectives JSONB,
    benefits JSONB,
    eligibility_criteria JSONB,
    application_process JSONB,
    required_documents JSONB,
    budget_allocation DECIMAL(15,2),
    start_date DATE,
    end_date DATE,
    application_deadline DATE,
    geographical_coverage JSONB,
    official_website TEXT,
    helpline_numbers JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Scheme applications
CREATE TABLE scheme_applications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    scheme_id UUID REFERENCES government_schemes(id) ON DELETE CASCADE,
    applicant_id UUID REFERENCES users(id) ON DELETE CASCADE,
    application_number VARCHAR(100) UNIQUE,
    application_data JSONB,  -- Complete application form data
    documents JSONB,  -- Array of uploaded document URLs
    status VARCHAR(50) DEFAULT 'draft',  -- draft, submitted, under_review, approved, rejected, benefited
    submission_date DATE,
    review_date DATE,
    approval_date DATE,
    rejection_reason TEXT,
    benefit_amount DECIMAL(12,2),
    benefit_received_date DATE,
    remarks TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### **9. Notification & Communication Schema**

```sql
-- Notification templates
CREATE TABLE notification_templates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    type VARCHAR(100),  -- email, sms, push, whatsapp
    category VARCHAR(100),  -- weather_alert, price_alert, consultation
    subject_template TEXT,
    body_template TEXT,
    variables JSONB,  -- Template variables
    language VARCHAR(10) DEFAULT 'en',
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- User notifications
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(100),
    title VARCHAR(500),
    message TEXT,
    data JSONB,  -- Additional notification data
    channels JSONB,  -- Array of delivery channels
    priority VARCHAR(20) DEFAULT 'normal',  -- low, normal, high, urgent
    scheduled_at TIMESTAMP WITH TIME ZONE,
    sent_at TIMESTAMP WITH TIME ZONE,
    read_at TIMESTAMP WITH TIME ZONE,
    action_taken BOOLEAN DEFAULT FALSE,
    status VARCHAR(50) DEFAULT 'pending',  -- pending, sent, failed, expired
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- User notification preferences
CREATE TABLE notification_preferences (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE UNIQUE,
    email_enabled BOOLEAN DEFAULT TRUE,
    sms_enabled BOOLEAN DEFAULT TRUE,
    push_enabled BOOLEAN DEFAULT TRUE,
    whatsapp_enabled BOOLEAN DEFAULT FALSE,
    weather_alerts BOOLEAN DEFAULT TRUE,
    price_alerts BOOLEAN DEFAULT TRUE,
    expert_messages BOOLEAN DEFAULT TRUE,
    marketing_messages BOOLEAN DEFAULT FALSE,
    system_updates BOOLEAN DEFAULT TRUE,
    preferred_language VARCHAR(10) DEFAULT 'en',
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

## 📊 ClickHouse Schema (Analytics Database)

```sql
-- User activity tracking
CREATE TABLE user_activities (
    id UUID,
    user_id UUID,
    activity_type String,
    activity_data String,  -- JSON data
    location Tuple(Float64, Float64),
    user_agent String,
    ip_address String,
    session_id String,
    timestamp DateTime64(3) DEFAULT now64()
) ENGINE = ReplacingMergeTree()
ORDER BY (user_id, timestamp)
PARTITION BY toYYYYMM(timestamp);

-- API usage metrics
CREATE TABLE api_metrics (
    request_id String,
    endpoint String,
    method String,
    user_id UUID,
    response_status UInt16,
    response_time_ms UInt32,
    request_size UInt32,
    response_size UInt32,
    timestamp DateTime64(3) DEFAULT now64()
) ENGINE = MergeTree()
ORDER BY (endpoint, timestamp)
PARTITION BY toYYYYMM(timestamp);

-- Agricultural data analytics
CREATE TABLE crop_analytics (
    id UUID,
    farmer_id UUID,
    crop_id UUID,
    field_id UUID,
    season String,
    crop_year UInt16,
    area Float32,
    yield Float32,
    production_cost Float32,
    revenue Float32,
    profit Float32,
    roi Float32,
    timestamp DateTime64(3) DEFAULT now64()
) ENGINE = ReplacingMergeTree()
ORDER BY (farmer_id, crop_id, timestamp)
PARTITION BY crop_year;

-- Weather data
CREATE TABLE weather_data (
    location_id String,
    coordinates Tuple(Float64, Float64),
    temperature Float32,
    humidity Float32,
    rainfall Float32,
    wind_speed Float32,
    wind_direction UInt16,
    pressure Float32,
    uv_index Float32,
    timestamp DateTime64(3) DEFAULT now64()
) ENGINE = MergeTree()
ORDER BY (location_id, timestamp)
PARTITION BY toYYYYMM(timestamp);

-- Market price trends
CREATE TABLE price_trends (
    crop_id UUID,
    market_id UUID,
    price Float32,
    volume Float32,
    trend String,
    source String,
    timestamp DateTime64(3) DEFAULT now64()
) ENGINE = ReplacingMergeTree()
ORDER BY (crop_id, market_id, timestamp)
PARTITION BY toYYYYMM(timestamp);
```

## 🔍 Elasticsearch Schema (Search Index)

```json
{
  "mappings": {
    "properties": {
      "content_type": { "type": "keyword" },
      "title": { 
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "content": { 
        "type": "text",
        "analyzer": "standard"
      },
      "tags": { "type": "keyword" },
      "category": { "type": "keyword" },
      "language": { "type": "keyword" },
      "location": { "type": "geo_point" },
      "created_at": { "type": "date" },
      "popularity_score": { "type": "float" },
      "user_ratings": { "type": "float" }
    }
  }
}
```

## 🗄️ Database Indexes & Performance

### **Primary Database Indexes**
```sql
-- User management indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_type ON users(user_type);
CREATE INDEX idx_user_profiles_location ON user_profiles USING GIST(location);

-- Crop management indexes
CREATE INDEX idx_crop_plans_farmer_season ON crop_plans(farmer_id, season, crop_year);
CREATE INDEX idx_crop_plans_status ON crop_plans(status);
CREATE INDEX idx_growth_records_plan_date ON crop_growth_records(crop_plan_id, record_date);

-- Market data indexes
CREATE INDEX idx_commodity_prices_date ON commodity_prices(price_date DESC);
CREATE INDEX idx_commodity_prices_crop_market ON commodity_prices(crop_id, market_id, price_date DESC);

-- Consultation indexes
CREATE INDEX idx_consultations_farmer ON consultations(farmer_id, created_at DESC);
CREATE INDEX idx_consultations_expert ON consultations(expert_id, created_at DESC);
CREATE INDEX idx_consultations_status ON consultations(status);

-- Equipment indexes
CREATE INDEX idx_equipment_category ON equipment(category_id);
CREATE INDEX idx_rental_bookings_dates ON rental_bookings(start_datetime, end_datetime);

-- Notification indexes
CREATE INDEX idx_notifications_user_status ON notifications(user_id, status, created_at DESC);
CREATE INDEX idx_notifications_scheduled ON notifications(scheduled_at) WHERE status = 'pending';
```

### **Partitioning Strategy**
```sql
-- Partition large tables by date
-- Example: Partition notifications by month
CREATE TABLE notifications_y2024m01 PARTITION OF notifications
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Partition analytics tables by year
CREATE TABLE crop_analytics_2024 PARTITION OF crop_analytics
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

## 🔄 Data Migration & Backup Strategy

### **Backup Configuration**
```sql
-- Daily full backup
pg_dump -h localhost -U agriguru -d agriguru_main -F c -b -v -f backup_$(date +%Y%m%d).backup

-- Point-in-time recovery setup
archive_mode = on
archive_command = 'cp %p /backup/wal_archive/%f'
wal_level = replica
```

### **Data Retention Policies**
- **User activity logs**: 2 years
- **API metrics**: 1 year
- **Weather data**: 5 years
- **Price data**: 10 years
- **Crop records**: Permanent
- **Consultation records**: 7 years
- **Transaction records**: 7 years (compliance requirement)

## 🔐 Security & Access Control

### **Row Level Security (RLS)**
```sql
-- Enable RLS for sensitive tables
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;

-- Farmers can only access their own data
CREATE POLICY farmer_own_data ON user_profiles
    FOR ALL TO farmer_role
    USING (user_id = current_user_id());

-- Experts can view anonymized farmer data for consultations
CREATE POLICY expert_consultation_access ON crop_plans
    FOR SELECT TO expert_role
    USING (id IN (SELECT crop_plan_id FROM consultations WHERE expert_id = current_user_id()));
```

### **Data Encryption**
- **At Rest**: PostgreSQL TDE (Transparent Data Encryption)
- **In Transit**: SSL/TLS encryption for all connections
- **PII Fields**: Application-level encryption using AES-256
- **Backup Encryption**: Encrypted backup files with separate key management

This comprehensive database schema provides a solid foundation for the AgriGuru platform, ensuring scalability, performance, and data integrity while supporting all the required functionalities.