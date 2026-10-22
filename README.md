# AI Fitness App

AI Fitness App is a full-stack fitness tracking platform built with Spring Boot microservices and a React frontend. Users authenticate with OAuth2/Keycloak, record activities, review their activity history, and receive AI-generated recommendations based on their fitness data.

## Features

- OAuth2 Authorization Code with PKCE authentication through Keycloak
- API gateway with JWT validation, CORS configuration, and service routing
- User registration and profile lookup
- Fitness activity tracking and activity history
- AI recommendations generated from activity events
- Service discovery with Netflix Eureka
- Centralized configuration with Spring Cloud Config Server
- PostgreSQL persistence for users
- MongoDB persistence for activities and recommendations
- RabbitMQ messaging between the activity and AI services
- React UI built with Vite, Material UI, Redux Toolkit, and Axios

## Architecture

```text
                            +----------------------+
                            |  React/Vite frontend  |
                            |    localhost:5173     |
                            +----------+-----------+
                                       |
                              OAuth2 + REST /api
                                       |
                            +----------v-----------+
                            |     API Gateway      |
                            |    localhost:8080    |
                            +----+---------+-------+
                                 |         |
                       Eureka LB |         | Eureka LB
                                 |         |
              +------------------v-+     +-v------------------+
              |   User Service     |     | Activity Service   |
              |   localhost:8081   |     |   localhost:8082   |
              |   PostgreSQL       |     |   MongoDB           |
              +--------------------+     +---------+-----------+
                                                    |
                                             RabbitMQ events
                                                    |
                                           +--------v---------+
                                           |    AI Service    |
                                           |   localhost:8083 |
                                           |   MongoDB/Gemini |
                                           +------------------+

       Config Server: localhost:8888       Eureka: localhost:8761
       Keycloak:      localhost:8181        PostgreSQL: localhost:5432
       MongoDB:       localhost:27017       RabbitMQ: localhost:5672
```

### Services

| Module | Port | Responsibility |
| --- | ---: | --- |
| `eureka` | `8761` | Service registry and discovery |
| `configserver` | `8888` | Loads shared configuration from `configserver/src/main/resources/config` |
| `gateway` | `8080` | JWT-protected entry point and route forwarding |
| `userservice` | `8081` | User registration, profile lookup, and validation |
| `activityservice` | `8082` | Stores and retrieves fitness activities |
| `aiservice` | `8083` | Consumes activity events and serves AI recommendations |
| `fitness-app-frontend` | `5173` | Browser client |

## Technology Stack

### Backend

- Java 17
- Spring Boot
- Spring Cloud Config
- Spring Cloud Gateway
- Spring Cloud Netflix Eureka
- Spring Security OAuth2 Resource Server
- Spring Data JPA
- Spring Data MongoDB
- Spring AMQP/RabbitMQ
- PostgreSQL
- MongoDB
- Maven

### Frontend

- React 19
- Vite
- React Router
- Material UI
- Redux Toolkit
- Axios
- `react-oauth2-code-pkce`

## Repository Structure

```text
.
├── activityservice/       Activity REST API and RabbitMQ publisher
├── aiservice/             Recommendation API, event listener, and Gemini integration
├── configserver/          Spring Cloud Config Server and service configuration
├── eureka/                Eureka service registry
├── gateway/               API gateway and JWT security configuration
├── userservice/           User REST API and PostgreSQL persistence
└── fitness-app-frontend/  React/Vite web application
```

## Prerequisites

Install the following before starting the application:

- JDK 17
- Node.js and npm
- PostgreSQL
- MongoDB
- RabbitMQ
- Keycloak
- A Gemini API key

The default configuration expects the supporting services to run locally:

| Dependency | Default address |
| --- | --- |
| PostgreSQL | `localhost:5432` |
| MongoDB | `localhost:27017` |
| RabbitMQ | `localhost:5672` |
| Keycloak | `localhost:8181` |

## Configuration

The Config Server uses the native profile and reads YAML files from:

```text
configserver/src/main/resources/config/
```

The checked-in configuration is intended for local development. Before deploying, replace local credentials and URLs with environment-specific values.

### PostgreSQL

Create the user database:

```sql
CREATE DATABASE fitness_user_db;
```

The user service uses PostgreSQL for JPA persistence. Its current local defaults are:

```text
Host:     localhost
Port:     5432
Database: fitness_user_db
Username: postgres
```

Do not use the development password from the repository in a shared or production environment.

### MongoDB

The services use these databases:

- Activity service: `fitnessactivity`
- AI service: `fitnessrecommendation`

MongoDB creates the databases and collections when the services first write data.

### RabbitMQ

The activity service publishes activity events through the `fitness.exchange` exchange using the `activity.tracking` routing key. The AI service listens on the `activity.queue` queue and creates or updates recommendations.

### Gemini

The AI service reads the Gemini integration values from environment variables:

```powershell
$env:GEMINI_API_URL = "https://your-gemini-endpoint"
$env:GEMINI_API_KEY = "your-api-key"
```

Set these variables in the shell that starts the Config Server or AI service, depending on how your configuration is supplied. Never commit API keys to source control.

### Keycloak

The gateway validates bearer tokens using the Keycloak realm:

```text
http://localhost:8181/realms/fitness-oauth2
```

The frontend is configured for:

```text
Client ID:     oauth2-pkce-client
Redirect URI:  http://localhost:5173
Scopes:        openid profile email offline_access
```

Create a Keycloak realm and client matching these values, or update `fitness-app-frontend/src/authConfig.js` and the gateway configuration for your environment. The client must allow the frontend redirect URI.

## Running the Application

Start the backend services in this order so that configuration and discovery are available before dependent services:

### 1. Start Eureka

```powershell
cd eureka
.\mvnw.cmd spring-boot:run
```

Open the Eureka dashboard at <http://localhost:8761>.

### 2. Start the Config Server

```powershell
cd configserver
.\mvnw.cmd spring-boot:run
```

The Config Server is available at <http://localhost:8888>.

### 3. Start the application services

Open a separate terminal for each service:

```powershell
cd userservice
.\mvnw.cmd spring-boot:run
```

```powershell
cd activityservice
.\mvnw.cmd spring-boot:run
```

```powershell
cd aiservice
.\mvnw.cmd spring-boot:run
```

```powershell
cd gateway
.\mvnw.cmd spring-boot:run
```

### 4. Start the frontend

```powershell
cd fitness-app-frontend
npm install
npm run dev
```

Open <http://localhost:5173> in a browser.

Useful frontend commands:

```powershell
npm run lint
npm run build
npm run preview
```

## API Reference

All application API calls should normally go through the gateway at `http://localhost:8080`.

### Users

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/api/users/register` | Register a user |
| `GET` | `/api/users/{userId}` | Get a user profile |
| `GET` | `/api/users/{userId}/validate` | Check whether a user exists |

### Activities

The activity endpoints require the `X-User-ID` header. The frontend adds this header from the authenticated user state.

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/api/activities` | Track an activity |
| `GET` | `/api/activities` | List activities for the current user |
| `GET` | `/api/activities/{activityId}` | Get one activity |

### Recommendations

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/api/recommendations/user/{userId}` | List recommendations for a user |
| `GET` | `/api/recommendations/activity/{activityId}` | Get the recommendation for an activity |

All gateway requests must include a valid bearer token issued by the configured Keycloak realm.

## Request Flow

1. The user signs in through Keycloak using the frontend’s PKCE flow.
2. The frontend stores the access token and sends it as a bearer token to the gateway.
3. The gateway validates the JWT and forwards the request to the service discovered through Eureka.
4. The activity service persists the activity in MongoDB and publishes an event to RabbitMQ.
5. The AI service consumes the event, calls Gemini, and stores the recommendation in MongoDB.
6. The frontend requests the recommendation through the gateway when viewing activity details.

## Testing

Each Spring Boot service includes a basic application context test. Run a service’s tests with:

```powershell
cd userservice
.\mvnw.cmd test
```

Repeat the command from `activityservice`, `aiservice`, `configserver`, `eureka`, and `gateway` as needed.

Run frontend linting and a production build with:

```powershell
cd fitness-app-frontend
npm run lint
npm run build
```

## Production Considerations

- Replace all local credentials with secret-managed configuration.
- Do not commit database, broker, Keycloak, or Gemini secrets.
- Use HTTPS and update Keycloak issuer/JWK URLs accordingly.
- Restrict gateway CORS origins to the deployed frontend.
- Configure durable RabbitMQ resources and production connection credentials.
- Use managed PostgreSQL and MongoDB instances or secure, persistent deployments.
- Replace `ddl-auto: update` with an explicit database migration strategy.
- Add health checks, metrics, centralized logs, and rate limiting before exposing the gateway publicly.

## License and Usage

This repository contains an educational fitness application. Review and comply with the licensing and usage terms that apply to the source materials and all third-party dependencies before redistributing or deploying it.
