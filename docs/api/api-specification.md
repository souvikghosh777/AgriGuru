# AgriGuru API Specification

## 📋 API Overview

The AgriGuru platform provides a comprehensive RESTful API that enables integration with agricultural management services. This document outlines all available endpoints, request/response formats, and integration guidelines.

### **Base URL**
- **Production**: `https://api.agriguru.com/v1`
- **Staging**: `https://staging-api.agriguru.com/v1`
- **Development**: `http://localhost:5000/api/v1`

### **Authentication**
All API requests require authentication using JWT tokens:
```
Authorization: Bearer <jwt_token>
```

### **Content Type**
All requests and responses use JSON format:
```
Content-Type: application/json
```

## 🔐 Authentication & Authorization

### **POST /auth/login**
Authenticate user and receive JWT token.

**Request:**
```json
{
  "username": "farmer123",
  "password": "securepassword",
  "user_type": "farmer"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user_123",
    "username": "farmer123",
    "role": "farmer",
    "profile": {
      "name": "John Farmer",
      "location": "Punjab, India",
      "farm_size": "10 acres"
    }
  },
  "expires_in": 3600
}
```

### **POST /auth/register**
Register new user account.

**Request:**
```json
{
  "username": "newfarmer",
  "email": "farmer@example.com",
  "password": "securepassword",
  "user_type": "farmer",
  "profile": {
    "name": "New Farmer",
    "phone": "+91-9876543210",
    "location": {
      "state": "Punjab",
      "district": "Ludhiana",
      "village": "Farmville"
    },
    "farm_details": {
      "size": "5 acres",
      "soil_type": "loamy",
      "water_source": "tubewell"
    }
  }
}
```

### **POST /auth/refresh**
Refresh expired JWT token.

### **POST /auth/logout**
Invalidate current session.

## 👥 User Management

### **GET /users/profile**
Get current user profile information.

### **PUT /users/profile**
Update user profile details.

### **GET /users/farmers**
List all farmers (Admin/Expert access only).

### **GET /users/experts**
List available agricultural experts.

## 🌱 Crop Management

### **GET /crops**
List all supported crops with basic information.

**Response:**
```json
{
  "crops": [
    {
      "id": "crop_wheat",
      "name": "Wheat",
      "scientific_name": "Triticum aestivum",
      "category": "cereal",
      "season": "rabi",
      "growth_duration": "120-150 days",
      "water_requirement": "medium",
      "soil_types": ["loamy", "clay-loam"]
    }
  ],
  "total": 50,
  "page": 1,
  "per_page": 20
}
```

### **POST /crops/plan**
Create new crop planning entry.

**Request:**
```json
{
  "crop_id": "crop_wheat",
  "field_id": "field_123",
  "planting_date": "2024-11-15",
  "area": "2.5 acres",
  "variety": "HD-2967",
  "expected_yield": "25 quintals",
  "notes": "Using organic fertilizers this season"
}
```

### **GET /crops/plans**
Get user's crop planning records.

### **PUT /crops/plans/{plan_id}**
Update existing crop plan.

### **POST /crops/growth-tracking**
Record crop growth stage and photos.

**Request:**
```json
{
  "plan_id": "plan_123",
  "growth_stage": "flowering",
  "date": "2024-12-15",
  "photos": ["base64_image_1", "base64_image_2"],
  "observations": "Healthy flowering, good plant height",
  "issues": []
}
```

### **GET /crops/harvest-prediction/{plan_id}**
Get AI-based harvest yield prediction.

**Response:**
```json
{
  "predicted_yield": "23.5 quintals",
  "confidence": 0.85,
  "harvest_date_range": {
    "earliest": "2025-03-10",
    "latest": "2025-03-20"
  },
  "factors": {
    "weather_impact": "positive",
    "soil_health": "good",
    "growth_progress": "on_track"
  }
}
```

## 🌤️ Weather Services

### **GET /weather/current**
Get current weather conditions for user's location.

**Parameters:**
- `lat`: Latitude (optional, uses user's default location)
- `lon`: Longitude (optional, uses user's default location)

**Response:**
```json
{
  "location": {
    "name": "Ludhiana, Punjab",
    "coordinates": [30.9010, 75.8573]
  },
  "current": {
    "temperature": 18.5,
    "humidity": 65,
    "wind_speed": 8.2,
    "wind_direction": "NW",
    "precipitation": 0,
    "pressure": 1013.2,
    "uv_index": 3,
    "visibility": 10
  },
  "agricultural_advice": {
    "irrigation_needed": false,
    "spray_conditions": "favorable",
    "field_work_suitability": "excellent"
  }
}
```

### **GET /weather/forecast**
Get 7-day weather forecast.

### **POST /weather/alerts/subscribe**
Subscribe to weather alerts for specific conditions.

**Request:**
```json
{
  "alert_types": ["rain", "frost", "heatwave", "wind"],
  "thresholds": {
    "rain": 10,
    "temperature_min": 5,
    "temperature_max": 40,
    "wind_speed": 25
  },
  "notification_methods": ["app", "sms", "whatsapp"]
}
```

## 🐛 Pest & Disease Management

### **POST /pest-disease/identify**
Identify pest or disease from crop images.

**Request:**
```json
{
  "images": ["base64_image_1", "base64_image_2"],
  "crop_type": "wheat",
  "symptoms": ["yellow_spots", "wilting"],
  "location": {
    "lat": 30.9010,
    "lon": 75.8573
  },
  "date_observed": "2024-12-15"
}
```

**Response:**
```json
{
  "identification": {
    "primary_diagnosis": {
      "type": "disease",
      "name": "Wheat Yellow Rust",
      "scientific_name": "Puccinia striiformis",
      "confidence": 0.92,
      "severity": "moderate"
    },
    "alternative_diagnoses": [
      {
        "name": "Wheat Leaf Rust",
        "confidence": 0.23
      }
    ]
  },
  "treatment_recommendations": [
    {
      "type": "chemical",
      "treatment": "Propiconazole 25% EC",
      "dosage": "0.1% solution",
      "application_method": "foliar_spray",
      "timing": "early_morning"
    },
    {
      "type": "organic",
      "treatment": "Neem oil spray",
      "dosage": "2ml per liter",
      "frequency": "weekly"
    }
  ],
  "prevention_tips": [
    "Ensure proper field drainage",
    "Avoid over-irrigation",
    "Use resistant varieties"
  ]
}
```

### **GET /pest-disease/library**
Browse pest and disease information library.

### **POST /pest-disease/outbreak-report**
Report pest/disease outbreak in the region.

### **GET /pest-disease/regional-alerts**
Get regional pest and disease alerts.

## 📈 Market & Pricing

### **GET /market/prices**
Get current market prices for agricultural commodities.

**Parameters:**
- `commodity`: Crop name (optional)
- `market`: Market name (optional)
- `state`: State name (optional)

**Response:**
```json
{
  "prices": [
    {
      "commodity": "Wheat",
      "variety": "HD-2967",
      "market": "Ludhiana Mandi",
      "state": "Punjab",
      "price": {
        "min": 2100,
        "max": 2150,
        "modal": 2125,
        "unit": "per_quintal",
        "currency": "INR"
      },
      "date": "2024-12-15",
      "trend": "stable",
      "change_percent": 0.5
    }
  ],
  "last_updated": "2024-12-15T10:30:00Z"
}
```

### **GET /market/trends/{commodity}**
Get price trends and forecasting for specific commodity.

### **POST /market/alerts**
Set price alerts for commodities.

### **GET /market/demand-forecast**
Get AI-based demand forecasting for crops.

## 👨‍🔬 Expert Consultation

### **GET /experts**
List available agricultural experts.

**Response:**
```json
{
  "experts": [
    {
      "id": "expert_123",
      "name": "Dr. Agricultural Expert",
      "specialization": ["crop_management", "pest_control"],
      "languages": ["hindi", "punjabi", "english"],
      "rating": 4.8,
      "experience_years": 15,
      "consultation_fee": 500,
      "availability": {
        "status": "available",
        "next_slot": "2024-12-16T14:00:00Z"
      }
    }
  ]
}
```

### **POST /consultations/book**
Book consultation with an expert.

### **GET /consultations/my-consultations**
Get user's consultation history.

### **POST /consultations/ai-chat**
Chat with AI agricultural assistant.

**Request:**
```json
{
  "message": "My wheat crop has yellow spots on leaves. What should I do?",
  "context": {
    "crop": "wheat",
    "location": "Punjab",
    "growth_stage": "tillering"
  },
  "language": "english"
}
```

**Response:**
```json
{
  "response": "Based on your description, the yellow spots on wheat leaves could indicate Yellow Rust disease. Here's what you should do:\n\n1. Immediate Action: Apply Propiconazole fungicide...",
  "confidence": 0.88,
  "follow_up_questions": [
    "Are the spots spreading rapidly?",
    "What is the current weather condition?",
    "Have you noticed any other symptoms?"
  ],
  "related_resources": [
    {
      "title": "Wheat Disease Management Guide",
      "url": "/resources/wheat-diseases"
    }
  ]
}
```

## 🚜 Equipment Management

### **GET /equipment/catalog**
Browse available farming equipment.

**Response:**
```json
{
  "equipment": [
    {
      "id": "eq_tractor_001",
      "name": "Mahindra 575 DI Tractor",
      "category": "tractor",
      "specifications": {
        "power": "45 HP",
        "fuel_type": "diesel",
        "transmission": "8F+2R"
      },
      "rental": {
        "available": true,
        "price_per_day": 1500,
        "price_per_hour": 250
      },
      "purchase": {
        "available": true,
        "price": 850000,
        "financing_available": true
      },
      "supplier": {
        "name": "Punjab Tractors",
        "rating": 4.5,
        "location": "Ludhiana"
      }
    }
  ]
}
```

### **POST /equipment/rental/book**
Book equipment for rental.

### **GET /equipment/rental/my-bookings**
Get user's equipment rental history.

### **POST /equipment/purchase/inquiry**
Submit purchase inquiry for equipment.

## 💳 Financial Services

### **GET /finance/loans**
Get available loan schemes.

### **POST /finance/loans/apply**
Apply for agricultural loan.

### **GET /finance/insurance**
Browse crop insurance options.

### **POST /payments/process**
Process payment transaction.

### **GET /subsidies/available**
Get government subsidies available to user.

## 🏛️ Government Schemes

### **GET /schemes**
List available government schemes.

**Response:**
```json
{
  "schemes": [
    {
      "id": "scheme_pmkisan",
      "name": "PM-KISAN",
      "description": "Income support to farmers",
      "benefit_amount": 6000,
      "benefit_period": "annual",
      "eligibility_criteria": [
        "Small and marginal farmers",
        "Landholding up to 2 hectares",
        "Indian citizen"
      ],
      "application_process": "online",
      "documents_required": [
        "Aadhaar card",
        "Bank account details",
        "Land ownership documents"
      ],
      "deadline": "2024-12-31"
    }
  ]
}
```

### **POST /schemes/apply**
Apply for government scheme.

### **GET /schemes/my-applications**
Track scheme application status.

### **POST /schemes/eligibility-check**
Check eligibility for schemes.

## 📊 Analytics & Reports

### **GET /analytics/farm-dashboard**
Get farm performance analytics.

**Response:**
```json
{
  "farm_summary": {
    "total_area": "10 acres",
    "crops_grown": 3,
    "current_season": "rabi",
    "expected_yield": "45 quintals",
    "estimated_revenue": 95000
  },
  "performance_metrics": {
    "yield_efficiency": 0.85,
    "cost_efficiency": 0.78,
    "profit_margin": 0.22,
    "sustainability_score": 0.71
  },
  "recent_activities": [
    {
      "date": "2024-12-10",
      "activity": "Fertilizer application",
      "crop": "Wheat",
      "field": "Field A"
    }
  ]
}
```

### **GET /analytics/crop-performance/{crop_id}**
Get detailed crop performance analytics.

### **GET /analytics/financial-summary**
Get financial performance summary.

### **POST /reports/generate**
Generate custom reports.

## 📱 Notification Services

### **POST /notifications/send**
Send notification to users.

### **GET /notifications**
Get user's notifications.

### **PUT /notifications/{id}/mark-read**
Mark notification as read.

### **POST /notifications/preferences**
Update notification preferences.

## 🔍 Search & Discovery

### **GET /search**
Search across platform content.

**Parameters:**
- `q`: Search query
- `type`: Content type (crops, experts, equipment, etc.)
- `location`: Location filter
- `category`: Category filter

**Response:**
```json
{
  "results": [
    {
      "type": "crop",
      "id": "crop_wheat",
      "title": "Wheat (Triticum aestivum)",
      "description": "Winter cereal crop suitable for northern India",
      "relevance_score": 0.95
    },
    {
      "type": "expert",
      "id": "expert_123",
      "title": "Dr. Wheat Specialist",
      "description": "15+ years experience in wheat cultivation",
      "relevance_score": 0.87
    }
  ],
  "total_results": 156,
  "search_time": "0.23s"
}
```

## 📊 API Status & Health

### **GET /health**
API health check endpoint.

### **GET /status**
Detailed API status information.

### **GET /metrics**
API performance metrics (Admin only).

## 🔧 API Usage Guidelines

### **Rate Limiting**
- **Free Tier**: 100 requests per hour
- **Farmer Plan**: 1,000 requests per hour
- **Professional Plan**: 10,000 requests per hour
- **Enterprise Plan**: Unlimited

### **Error Handling**
All API errors follow a consistent format:

```json
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Username or password is incorrect",
    "details": "Please check your credentials and try again",
    "timestamp": "2024-12-15T10:30:00Z",
    "request_id": "req_abc123"
  }
}
```

### **Common HTTP Status Codes**
- `200 OK`: Request successful
- `201 Created`: Resource created successfully
- `400 Bad Request`: Invalid request parameters
- `401 Unauthorized`: Authentication required
- `403 Forbidden`: Insufficient permissions
- `404 Not Found`: Resource not found
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error

### **Pagination**
List endpoints support pagination:

```json
{
  "data": [...],
  "pagination": {
    "current_page": 1,
    "per_page": 20,
    "total_pages": 5,
    "total_items": 100,
    "has_next": true,
    "has_previous": false
  }
}
```

### **Filtering & Sorting**
Most list endpoints support filtering and sorting:
- `?filter[field]=value`: Filter by field value
- `?sort=field`: Sort by field (ascending)
- `?sort=-field`: Sort by field (descending)
- `?search=query`: Search within results

### **API Versioning**
- Current version: `v1`
- Version specified in URL: `/api/v1/`
- Backward compatibility maintained for 12 months
- Deprecation notices provided 6 months in advance

## 🔒 Security Considerations

### **Authentication Security**
- JWT tokens expire after 1 hour
- Refresh tokens expire after 30 days
- Password requirements: minimum 8 characters, mixed case, numbers
- Account lockout after 5 failed attempts

### **Data Protection**
- All API communications use HTTPS
- Sensitive data encrypted at rest
- PII data access logged and audited
- GDPR compliance for EU users

### **API Security**
- Rate limiting prevents abuse
- SQL injection protection
- XSS prevention in responses
- CORS configuration for web clients
- API key rotation capability