# Design Document: Civic Engagement Platform

## Overview

The Civic Engagement Platform is a cloud-native system built entirely with Flutter that connects verified citizens and volunteers with local civic opportunities. The platform consists of two applications:

1. **Flutter Mobile App**: For citizens and volunteers to discover opportunities, report problems, track volunteer hours, and receive notifications
2. **Flutter Web Dashboard**: For government officials to manage tasks, review reports, monitor participation, and analyze civic engagement metrics

Both applications share a common backend API layer built on AWS serverless services, ensuring scalability, security, and reliability while integrating with Indian government systems for identity verification.

## Architecture

The platform follows a serverless-first architecture with Flutter applications for both mobile and web, sharing a common backend API layer.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer                             │
├──────────────────────────┬──────────────────────────────────┤
│   Flutter Mobile App     │   Flutter Web Dashboard          │
│   - Android & iOS        │   - Desktop Browsers             │
│   - Citizens/Volunteers  │   - Government Officials         │
│   - BLoC State Mgmt      │   - BLoC State Mgmt              │
└──────────────────────────┴──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              AWS API Gateway                                 │
│  - REST APIs                                                 │
│  - Rate limiting & throttling                               │
│  - JWT token validation                                     │
└─────────────────────────────────────────────────────────────┘
                           │
        ┌──────────────────┴──────────────────┐
        ▼                                      ▼
┌──────────────────┐              ┌──────────────────────┐
│ AWS Cognito      │              │ AWS Lambda           │
│ - Citizen Auth   │              │ - User Service       │
│ - Gov Auth       │              │ - Task Service       │
│ - JWT Tokens     │              │ - Matching Service   │
└──────────────────┘              │ - Reward Service     │
                                  └──────────────────────┘
                                           │
                                           ▼
                                  ┌──────────────────────┐
                                  │ Data Layer           │
                                  │ - DynamoDB           │
                                  │ - S3 (Images)        │
                                  │ - Bedrock (AI)       │
                                  └──────────────────────┘
```

## Technology Stack

### Frontend Applications
- **Framework**: Flutter (Dart)
- **State Management**: flutter_bloc (BLoC pattern)
- **HTTP Client**: dio
- **Local Storage**: hive, shared_preferences
- **Location Services**: geolocator, geocoding
- **Maps**: google_maps_flutter
- **Image Handling**: image_picker, cached_network_image
- **Notifications**: firebase_messaging, flutter_local_notifications
- **Authentication**: firebase_auth

### Backend Services
- **API Gateway**: AWS API Gateway (REST)
- **Compute**: AWS Lambda (Node.js/Python)
- **Authentication**: AWS Cognito + Firebase Auth
- **Database**: Amazon DynamoDB
- **Storage**: Amazon S3
- **AI Services**: Amazon Bedrock
- **Notifications**: Firebase Cloud Messaging
- **Identity Verification**: Sandbox.co.in Aadhaar API

## Components and Interfaces

### 1. Flutter Mobile App (Citizens/Volunteers)

**Purpose**: Mobile application for citizens to engage with civic opportunities.

**Key Features**:
- User registration with Aadhaar verification
- AI-powered skill profile management
- Problem reporting with camera integration
- Task discovery with map view
- Geolocation-based task check-in
- Volunteer hours tracking and rewards
- Push notifications
- Offline data caching

**Technology Stack**:
- Flutter SDK (latest stable)
- BLoC pattern for state management
- GetIt for dependency injection
- Clean Architecture (presentation → domain ← data)

**Key Screens**:
```
/login
/register
/verify-aadhaar
/dashboard
/profile
/tasks
/task-details/:id
/report-issue
/my-reports
/rewards
/notifications
/settings
```

**State Management Structure**:
```
features/
├── auth/
│   ├── presentation/bloc/auth_bloc.dart
│   ├── domain/usecases/
│   └── data/repositories/
├── profile/
│   ├── presentation/bloc/profile_bloc.dart
│   ├── domain/usecases/
│   └── data/repositories/
├── tasks/
│   ├── presentation/bloc/task_bloc.dart
│   ├── domain/usecases/
│   └── data/repositories/
├── reports/
│   ├── presentation/bloc/report_bloc.dart
│   ├── domain/usecases/
│   └── data/repositories/
└── rewards/
    ├── presentation/bloc/reward_bloc.dart
    ├── domain/usecases/
    └── data/repositories/
```

### 2. Flutter Web Dashboard (Government Officials)

**Purpose**: Web dashboard for government officials to manage civic engagement initiatives.

**Key Features**:
- Secure government authentication
- Problem reports review and management
- Task creation and assignment
- Volunteer verification and approval
- Analytics dashboards with charts
- Geographic heat maps
- Real-time statistics
- Export functionality (PDF, CSV)

**Technology Stack**:
- Flutter Web (latest stable)
- BLoC pattern for state management
- fl_chart for data visualization
- GetIt for dependency injection
- Clean Architecture

**Key Screens**:
```
/gov/login
/gov/dashboard
/gov/reports
/gov/reports/:id
/gov/tasks
/gov/tasks/create
/gov/tasks/:id
/gov/analytics
/gov/volunteers
/gov/verification
/gov/settings
```

**Responsive Design**:
- Optimized for desktop browsers (1280px and above)
- Sidebar navigation
- Data tables with sorting and filtering
- Modal dialogs for quick actions
- Keyboard shortcuts for efficiency

### 3. Authentication Service

**Purpose**: Manages dual authentication system for citizens and government officials.

**Implementation**:
- AWS Cognito for user pool management
- Firebase Auth for mobile app authentication
- JWT tokens for API authorization
- Role-based access control (RBAC)

**API Endpoints**:
```
POST /api/v1/auth/citizen/register
POST /api/v1/auth/citizen/login
POST /api/v1/auth/citizen/verify-aadhaar
POST /api/v1/auth/government/login
POST /api/v1/auth/refresh-token
POST /api/v1/auth/logout
GET /api/v1/auth/session
```

**Authentication Flow (Citizen)**:
1. User enters phone number in mobile app
2. OTP sent via SMS
3. User verifies OTP
4. User completes Aadhaar verification
5. JWT token issued with citizen role
6. Token stored securely in device
7. Subsequent API requests include token in Authorization header

**Authentication Flow (Government)**:
1. Official enters credentials in web dashboard
2. MFA challenge sent (SMS/Email)
3. Upon successful MFA, JWT token issued with government role
4. Token includes specific permissions
5. All actions logged for audit trail

### 4. Identity Verification Service

**Purpose**: Manages citizen identity verification through Aadhaar integration.

**Implementation**:
- AWS Lambda function for Aadhaar API integration
- Sandbox.co.in API for verification
- DynamoDB for storing verification status
- Retry logic with exponential backoff

**API Endpoints**:
```
POST /api/v1/identity/verify-aadhaar
GET /api/v1/identity/profile/{userId}
PUT /api/v1/identity/profile/{userId}
GET /api/v1/identity/verification-status/{userId}
```

**Verification Process**:
1. User enters Aadhaar number in mobile app
2. Lambda function calls Sandbox.co.in API
3. OTP sent to Aadhaar-registered mobile
4. User enters OTP
5. Verification status stored in DynamoDB
6. Profile marked as verified
7. Push notification sent to user

### 5. AI Skill Profiler Service

**Purpose**: Converts citizen text descriptions into structured skill tags using Amazon Bedrock.

**Implementation**:
- AWS Lambda function for text processing
- Amazon Bedrock (Claude model) for skill extraction
- DynamoDB for storing skill tags
- Caching layer for frequently used skills

**API Endpoints**:
```
POST /api/v1/skills/analyze
GET /api/v1/skills/tags/{userId}
PUT /api/v1/skills/tags/{userId}
GET /api/v1/skills/taxonomy
```

**Bedrock Prompt Template**:
```
Analyze the following personal description and extract relevant skill tags. 
Focus on professional skills, educational background, and areas of expertise.
Return a JSON array of skill tags with confidence scores.

Description: {user_description}

Expected format:
{
  "skills": [
    {"tag": "Veterinary Student", "confidence": 0.95},
    {"tag": "Animal Care", "confidence": 0.88}
  ]
}
```

**Mobile App Integration**:
- User enters bio in profile screen
- Real-time skill analysis as user types (debounced)
- Display suggested skills with confidence indicators
- User can accept, reject, or modify skills
- Visual skill tags displayed on profile

### 6. Problem Reporting Service

**Purpose**: Enables citizens to report local problems with photo evidence.

**Implementation**:
- AWS Lambda for report processing
- S3 for photo storage with compression
- DynamoDB for report metadata
- SNS for notification delivery

**API Endpoints**:
```
POST /api/v1/reports/submit
GET /api/v1/reports/{reportId}
GET /api/v1/reports/my-reports
GET /api/v1/reports/pending (government)
PUT /api/v1/reports/{reportId}/convert-to-task (government)
PUT /api/v1/reports/{reportId}/status (government)
```

**Mobile App Features**:
- Camera integration with image_picker
- Photo preview and editing
- Location auto-detection
- Category selection
- Offline submission queue
- Upload progress indicator
- Report status tracking

**Web Dashboard Features**:
- Filterable table of all reports
- Photo gallery view
- Map view showing report locations
- One-click task conversion
- Status update workflow
- Priority assignment

### 7. Task Management Service

**Purpose**: Manages civic engagement tasks from creation to completion.

**Implementation**:
- AWS Lambda for task operations
- DynamoDB for task storage
- S3 for completion evidence photos
- EventBridge for task lifecycle events

**API Endpoints**:
```
POST /api/v1/tasks/create (government)
GET /api/v1/tasks/available
GET /api/v1/tasks/{taskId}
POST /api/v1/tasks/{taskId}/start
POST /api/v1/tasks/{taskId}/complete
PUT /api/v1/tasks/{taskId}/verify (government)
GET /api/v1/tasks/my-tasks
GET /api/v1/tasks/all (government)
```

**Mobile App Task Features**:
- Browse available tasks with filters
- Interactive map showing nearby tasks
- Task details with requirements
- Geolocation-based "Start Task" button
- Photo upload for completion evidence
- Real-time distance calculation

**Web Dashboard Task Features**:
- Task creation form with skill selection
- Task assignment to volunteers
- Completion verification interface
- Task analytics and completion rates
- Bulk task operations

### 8. Matching and Notification Service

**Purpose**: Matches citizens with relevant civic opportunities and sends notifications.

**Implementation**:
- AWS Lambda for matching algorithm
- DynamoDB for user preferences
- Firebase Cloud Messaging for push notifications
- SNS for email notifications

**Matching Algorithm**:
```
match_score = (
    skill_overlap_score * 0.4 +
    location_proximity_score * 0.3 +
    availability_score * 0.2 +
    historical_engagement_score * 0.1
)
```

**API Endpoints**:
```
POST /api/v1/matching/find-candidates (government)
GET /api/v1/matching/recommendations/{userId}
POST /api/v1/notifications/send
GET /api/v1/notifications/preferences/{userId}
PUT /api/v1/notifications/preferences/{userId}
GET /api/v1/notifications/history/{userId}
```

**Mobile App Notification Features**:
- Push notifications with firebase_messaging
- In-app notification center
- Notification badge with count
- Mark as read/unread
- Notification preferences
- Deep linking to relevant screens

**Notification Types**:
- New task matches
- Report status updates
- Task reminders
- Reward achievements
- System announcements

### 9. Geofencing and Location Service

**Purpose**: Manages location-based task verification and geofenced check-ins.

**Implementation**:
- geolocator package for GPS access
- AWS Lambda for geofence validation
- DynamoDB for geofence definitions
- google_maps_flutter for map display

**API Endpoints**:
```
POST /api/v1/location/create-geofence (government)
POST /api/v1/location/verify-checkin
POST /api/v1/location/start-task
GET /api/v1/location/distance-to-task/{taskId}
GET /api/v1/location/nearby-tasks
```

**Geofence Configuration**:
- Radius: 100 meters from task coordinates
- Shape: Circular boundary
- Tolerance: 10-meter buffer for GPS accuracy
- Verification: Real-time GPS check

**Mobile App Map Features**:
- Interactive map with google_maps_flutter
- Current location marker
- Task markers with color-coded status
- Distance display to each task
- Route directions
- Geofence visualization

### 10. Automated Reward Service

**Purpose**: Tracks volunteer hours and automatically awards citizens upon task completion.

**Implementation**:
- AWS Lambda for verification logic
- Amazon Bedrock for AI-powered completion verification
- DynamoDB for reward records
- S3 for certificate generation

**API Endpoints**:
```
POST /api/v1/rewards/verify-completion
GET /api/v1/rewards/hours/{userId}
GET /api/v1/rewards/achievements/{userId}
GET /api/v1/rewards/certificate/{userId}
POST /api/v1/rewards/manual-verification (government)
GET /api/v1/rewards/leaderboard
```

**Verification Process**:
1. Citizen submits completion evidence (photo, description)
2. Lambda function calls Bedrock for AI verification
3. If verification passes, hours automatically awarded
4. If verification fails, task flagged for manual review
5. Achievement system updates
6. Push notification sent to citizen

**Mobile App Rewards Features**:
- Total volunteer hours display
- Progress bars for achievements
- Badge collection
- Activity timeline
- Downloadable certificates
- Leaderboard view

**Web Dashboard Verification**:
- Queue of tasks pending manual verification
- Side-by-side view of requirements and evidence
- Approve/reject buttons
- Bulk verification actions
- Verification history

### 11. Analytics and Reporting Service

**Purpose**: Provides comprehensive analytics for government officials.

**Implementation**:
- AWS Lambda for data aggregation
- DynamoDB for metrics storage
- QuickSight for advanced analytics (optional)
- fl_chart package for Flutter charts

**API Endpoints**:
```
GET /api/v1/analytics/overview (government)
GET /api/v1/analytics/participation-trends (government)
GET /api/v1/analytics/geographic-distribution (government)
GET /api/v1/analytics/task-completion-rates (government)
POST /api/v1/analytics/export (government)
```

**Web Dashboard Analytics**:
- Real-time statistics cards
- Line charts for participation trends
- Bar charts for task categories
- Geographic heat map
- Volunteer leaderboard
- Custom date range selector
- Export to PDF/CSV

**Available Metrics**:
- Total registered citizens
- Active volunteers
- Tasks created vs completed
- Average completion time
- Volunteer hours by category
- Geographic distribution
- Report response time

## Data Models

### Citizen Profile
```dart
class CitizenProfile {
  final String userId;
  final String userType; // "citizen"
  final bool aadhaarVerified;
  final DateTime verificationTimestamp;
  final PersonalInfo personalInfo;
  final List<SkillTag> skillTags;
  final NotificationPreferences preferences;
  final VolunteerStats volunteerStats;
  final DateTime createdAt;
  final DateTime lastLoginAt;
}

class SkillTag {
  final String tag;
  final double confidence;
  final DateTime extractedAt;
  final bool userModified;
}

class VolunteerStats {
  final double totalHours;
  final int tasksCompleted;
  final List<String> achievements;
  final int level;
  final int rank;
}
```

### Government Official Profile
```dart
class GovernmentProfile {
  final String userId;
  final String userType; // "government"
  final String governmentId;
  final PersonalInfo personalInfo;
  final String department;
  final String designation;
  final List<String> permissions;
  final bool mfaEnabled;
  final DateTime createdAt;
  final DateTime lastLoginAt;
}
```

### Task
```dart
class Task {
  final String taskId;
  final String title;
  final String description;
  final String category;
  final TaskLocation location;
  final List<String> requiredSkills;
  final double estimatedHours;
  final String priority; // "low", "medium", "high"
  final String status; // "open", "assigned", "in_progress", "completed", "verified"
  final String createdBy;
  final String? assignedTo;
  final DateTime createdAt;
  final DateTime? startedAt;
  final DateTime? completedAt;
  final DateTime? verifiedAt;
  final VerificationEvidence? verificationEvidence;
  final CheckInLocation? checkInLocation;
}

class TaskLocation {
  final double latitude;
  final double longitude;
  final String address;
  final String geofenceId;
}
```

### Problem Report
```dart
class ProblemReport {
  final String reportId;
  final String citizenId;
  final String title;
  final String description;
  final List<String> photos; // S3 URLs
  final ReportLocation location;
  final String category;
  final String priority;
  final String status; // "submitted", "under_review", "converted", "rejected"
  final DateTime submittedAt;
  final String? reviewedBy;
  final DateTime? reviewedAt;
  final String? convertedToTaskId;
  final String? rejectionReason;
}
```

### Notification
```dart
class Notification {
  final String notificationId;
  final String userId;
  final String type; // "task_match", "report_update", "reward", "announcement"
  final String title;
  final String message;
  final String? actionUrl;
  final bool isRead;
  final DateTime createdAt;
  final DateTime? readAt;
  final Map<String, dynamic>? metadata;
}
```

## User Flows

### Citizen Registration and Verification Flow
1. User downloads and opens mobile app
2. Taps "Register" button
3. Enters phone number
4. Receives and verifies OTP
5. Completes profile information
6. Initiates Aadhaar verification
7. Enters Aadhaar number
8. Receives Aadhaar OTP
9. Verifies Aadhaar OTP
10. Profile marked as verified
11. Redirected to dashboard

### Task Discovery and Completion Flow
1. User opens mobile app
2. Views dashboard with recommended tasks
3. Taps "Browse Tasks"
4. Filters by location and skills
5. Taps on task to view details
6. Sees task location on map
7. Taps "Start Task" button
8. App requests location permission
9. System verifies user is within 100m
10. Task status changes to "In Progress"
11. User completes task and takes photos
12. Submits completion with description
13. AI verification runs automatically
14. Volunteer hours added
15. Achievement notification displayed

### Problem Reporting Flow
1. User taps "Report Problem" in mobile app
2. Takes photo with camera or selects from gallery
3. Enters problem description
4. Selects category
5. Location auto-detected
6. Reviews report
7. Submits report
8. Receives confirmation
9. Can track status in "My Reports"

### Government Task Creation Flow
1. Official logs into web dashboard
2. Views "Problem Reports"
3. Filters and reviews reports
4. Clicks "Convert to Task"
5. Task form pre-filled with report data
6. Adds required skills and hours
7. Sets priority
8. Submits task
9. System matches with volunteers
10. Notifications sent to matched citizens

## Security Considerations

### Data Protection
- All data encrypted at rest (DynamoDB encryption)
- All data encrypted in transit (HTTPS/TLS)
- Aadhaar data stored in compliance with regulations
- Regular security audits

### Authentication & Authorization
- JWT tokens with expiration
- Role-based access control (RBAC)
- MFA for government officials
- Session timeout after 30 minutes

### API Security
- Rate limiting on API Gateway
- Request throttling
- Input validation and sanitization
- SQL injection prevention (NoSQL database)
- XSS protection

### Mobile App Security
- Secure storage for tokens (flutter_secure_storage)
- Certificate pinning for API calls
- Biometric authentication option
- Jailbreak/root detection

## Performance Considerations

### Mobile App
- Lazy loading for lists
- Image caching with cached_network_image
- Offline data caching with Hive
- Pagination for large datasets
- Debouncing for search inputs

### Web Dashboard
- Virtual scrolling for large tables
- Chart data aggregation
- Lazy loading for analytics
- Efficient state management with BLoC

### Backend
- DynamoDB auto-scaling
- Lambda concurrency limits
- S3 CloudFront CDN for images
- API Gateway caching
- Database query optimization

## Deployment Architecture

### Mobile App
- Android: Google Play Store
- iOS: Apple App Store
- CI/CD: GitHub Actions + Fastlane
- Beta testing: Firebase App Distribution

### Web Dashboard
- Hosting: AWS S3 + CloudFront
- CI/CD: GitHub Actions
- Domain: Custom domain with Route53
- SSL: AWS Certificate Manager

### Backend
- Infrastructure as Code: AWS CDK/Terraform
- Environment: Dev, Staging, Production
- Monitoring: CloudWatch
- Logging: CloudWatch Logs
- Alerting: SNS + CloudWatch Alarms

## Future Enhancements (Out of Scope)

- Multi-language support
- Offline mode for mobile app
- Real-time chat between citizens and officials
- Video upload support
- Social media integration
- Payment processing
- Third-party integrations
- Advanced AI features
