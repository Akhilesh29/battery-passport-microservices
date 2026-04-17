# Battery Passport Microservices

This repository contains a simplified microservices-based backend system for a digital battery passport platform. The stack includes Node.js, Express, MongoDB, Kafka, JWT authentication, and MinIO as an S3-compatible object store.

The repository supports two deployment modes:

- `Full local stack`: Docker Compose with MongoDB, Kafka, MinIO, and all four services
- `Hosted API-only demo`: three web APIs for free-tier hosting, with Mongo-backed document storage and mocked event handling instead of Kafka

## Service Descriptions

- `auth-service`: user registration, login, JWT issuance, and internal token verification
- `data-access-service`: CRUD for battery passports with role-based access and Kafka event publishing
- `document-service`: file upload, metadata management, and signed download links via MinIO
- `notification-service`: Kafka consumer that logs notification messages to text files

## Deployed URLs

### Railway API-Only Deployment

- Auth API: `https://auth-api-production-9405.up.railway.app`
- Data API: `https://data-api-production-7f79.up.railway.app`
- Document API: `https://document-api-production-1f70.up.railway.app`

### Health Endpoints

- Auth health: `https://auth-api-production-9405.up.railway.app/health`
- Data health: `https://data-api-production-7f79.up.railway.app/health`
- Document health: `https://document-api-production-1f70.up.railway.app/health`

## Setup Instructions

### Docker + `.env`

Use this mode if you want the complete local microservices stack with MongoDB, Kafka, MinIO, and the notification worker.

1. Clone the repository
2. Copy `.env.example` to `.env`
3. Review or update the values in `.env`
4. Make sure Docker Desktop is running
5. Start everything with:

```bash
docker compose up --build
```

6. Stop everything with:

```bash
docker compose down
```

### Local Services Started By Docker Compose

- `auth-service` on port `3001`
- `data-access-service` on port `3002`
- `document-service` on port `3003`
- `notification-service` on port `3004`
- `mongo` on port `27017`
- `kafka` on port `9092`
- `minio` on port `9000`
- `minio console` on port `9001`

### Local Development Notes

- The full local setup uses `Kafka` and `MinIO`
- The hosted Railway deployment does not use Kafka or MinIO
- For local testing, the default `.env.example` is already aligned with `docker-compose.yml`

### Hosted API-Only Setup

Use this mode if you want the lighter hosted version with only the public HTTP APIs.

Requirements:

- MongoDB Atlas cluster
- one shared `JWT_SECRET`
- three database URIs:
  - `auth_db`
  - `passport_db`
  - `document_db`

Hosted mode behavior:

- `KAFKA_DISABLED=true`
- `STORAGE_PROVIDER=mongo`
- documents are stored in MongoDB instead of MinIO
- notifications are mocked through logs rather than a separate worker service

### Required Environment Variables

The included `.env.example` already contains working defaults for local Docker usage.

- `JWT_SECRET`: secret used to sign and verify JWTs
- `AUTH_PORT`, `PASSPORT_PORT`, `DOCUMENT_PORT`, `NOTIFICATION_PORT`: service ports
- `MONGO_ROOT_USERNAME`, `MONGO_ROOT_PASSWORD`, `MONGO_HOST`, `MONGO_PORT`: MongoDB connection settings
- `AUTH_DB`, `PASSPORT_DB`, `DOCUMENT_DB`: per-service MongoDB database names
- `KAFKA_BROKER`, `KAFKA_CLIENT_ID`, `KAFKA_GROUP_ID`, `KAFKA_TOPIC_PASSPORT_EVENTS`: Kafka settings
- `KAFKA_DISABLED`: disables Kafka publishing and falls back to local mock event logging
- `MINIO_ENDPOINT`, `MINIO_PORT`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `MINIO_BUCKET`, `MINIO_USE_SSL`: S3-compatible storage settings
- `STORAGE_PROVIDER`: `minio` for S3-compatible storage or `mongo` for hosted API-only mode
- `AUTH_SERVICE_URL`: internal auth verification URL used by other services
- `NOTIFICATION_OUTPUT_DIR`: directory used by the notification service for log files

## Default Ports

- Auth Service: `3001`
- Data Access Service: `3002`
- Document Service: `3003`
- Notification Service: `3004`
- MongoDB: `27017`
- Kafka: `9092`
- MinIO API: `9000`
- MinIO Console: `9001`

## Internal Verification Flow

The data-access and document services validate bearer tokens by calling `POST /api/auth/verify` on the auth service over HTTP using the internal Docker service name.

## API Usage

### Auth Service

Base path: `/api/auth`

Railway base URL:

`https://auth-api-production-9405.up.railway.app/api/auth`

#### Register

`POST /api/auth/register`

```json
{
  "email": "admin@example.com",
  "password": "password123",
  "role": "admin"
}
```

#### Login

`POST /api/auth/login`

```json
{
  "email": "admin@example.com",
  "password": "password123"
}
```

The login response returns a JWT token. Use it as:

```http
Authorization: Bearer <token>
```

### Battery Passport Service

Base path: `/api/passports`

Railway base URL:

`https://data-api-production-7f79.up.railway.app/api/passports`

#### Create Passport

`POST /api/passports`

Admin only.

```json
{
  "data": {
    "generalInformation": {
      "batteryIdentifier": "BP-2024-011",
      "batteryModel": {
        "id": "LM3-BAT-2024",
        "modelName": "GMC WZX1"
      },
      "batteryMass": 450,
      "batteryCategory": "EV",
      "batteryStatus": "Original",
      "manufacturingDate": "2024-01-15",
      "manufacturingPlace": "Gigafactory Nevada",
      "warrantyPeriod": "8",
      "manufacturerInformation": {
        "manufacturerName": "Tesla Inc",
        "manufacturerIdentifier": "TESLA-001"
      }
    },
    "materialComposition": {
      "batteryChemistry": "LiFePO4",
      "criticalRawMaterials": [
        "Lithium",
        "Iron"
      ],
      "hazardousSubstances": [
        {
          "substanceName": "Lithium Hexafluorophosphate",
          "chemicalFormula": "LiPF6",
          "casNumber": "21324-40-3"
        }
      ]
    },
    "carbonFootprint": {
      "totalCarbonFootprint": 850,
      "measurementUnit": "kg CO2e",
      "methodology": "Life Cycle Assessment (LCA)"
    }
  }
}
```

#### Get Passport

`GET /api/passports/:id`

Admin and user roles are allowed.

#### Update Passport

`PUT /api/passports/:id`

Admin only. Uses the same request structure as create.

#### Delete Passport

`DELETE /api/passports/:id`

Admin only.

### Document Service

Base path: `/api/documents`

Railway base URL:

`https://document-api-production-1f70.up.railway.app/api/documents`

#### Upload File

`POST /api/documents/upload`

Use `multipart/form-data` with a `file` field.

Example response:

```json
{
  "docId": "string",
  "fileName": "battery-report.pdf",
  "createdAt": "2026-04-17T10:00:00.000Z"
}
```

#### Update File Metadata

`PUT /api/documents/:docId`

```json
{
  "metadata": {
    "documentType": "compliance-report",
    "passportId": "661f1b1f7b2f0e0012abcd34"
  }
}
```

#### Delete File

`DELETE /api/documents/:docId`

#### Get Download Link

`GET /api/documents/:docId`

Returns a signed MinIO download URL.

When `STORAGE_PROVIDER=mongo`, this endpoint returns an API download URL instead.

#### Download File In Mongo Storage Mode

`GET /api/documents/:docId/download`

This endpoint is used by the hosted API-only mode when documents are stored directly in MongoDB instead of MinIO.

## Kafka Topics And Payload Structure

### Topic

- `passport-events`

### Produced Events

- `passport.created`
- `passport.updated`
- `passport.deleted`

### Event Payload

The data-access service publishes JSON messages in this shape:

```json
{
  "eventType": "passport.created",
  "passportId": "661f1b1f7b2f0e0012abcd34",
  "batteryIdentifier": "BP-2024-011",
  "actor": {
    "id": "661f19d27b2f0e0012abcd12",
    "email": "admin@example.com",
    "role": "admin"
  },
  "occurredAt": "2026-04-17T10:00:00.000Z"
}
```

### Consumer Behavior

The notification service subscribes to `passport-events`, filters `passport.*` events, and writes notification output to text files in the configured logs directory.

When `KAFKA_DISABLED=true`, the data-access service logs the same event payload locally instead of publishing to Kafka. This is the behavior used by the free hosted API-only deployment.

## Render Deployment

This repository now includes a [`render.yaml`](./render.yaml) Blueprint for a Render-based deployment path.

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/Akhilesh29/Microservices-Backend-for-Battery-Passport-Platform)

Important notes:

- Render is a better fit than Vercel for this project because it supports long-running containerized services.
- The Blueprint provisions custom MongoDB, Kafka, and MinIO services plus the four Node services.
- Persistent disks on Render require paid service plans.
- Render Blueprint files do not support variable interpolation, so the application also supports split host and port environment variables for internal service discovery.

## Render Free API-Only Deployment

This repository also includes [`render-free.yaml`](./render-free.yaml) for a free-tier hosted demo deployment that exposes only the HTTP APIs:

- `battery-passport-auth-api`
- `battery-passport-data-api`
- `battery-passport-document-api`

This mode is designed for free hosting and makes these tradeoffs:

- Kafka is disabled with `KAFKA_DISABLED=true`
- Notifications are mocked by logging events in the data-access service
- Document files are stored directly in MongoDB with `STORAGE_PROVIDER=mongo`
- The standalone notification worker is not deployed
- MinIO is not deployed

### What you need for the free hosted mode

- A free MongoDB Atlas cluster
- Three MongoDB connection strings, one per service database:
  - auth service: `.../auth_db`
  - passport service: `.../passport_db`
  - document service: `.../document_db`
- A shared `JWT_SECRET`

### Deploying the free hosted mode on Render

1. In Render, create a new Blueprint deployment from this repo.
2. Select `render-free.yaml`.
3. When prompted, provide:
   - `JWT_SECRET`
   - `MONGO_URI` for `battery-passport-auth-api`
   - `MONGO_URI` for `battery-passport-data-api`
   - `MONGO_URI` for `battery-passport-document-api`
4. Finish the Blueprint creation flow.

This free mode is the simplest way to host the APIs publicly without paid private services, workers, Kafka, or MinIO.
