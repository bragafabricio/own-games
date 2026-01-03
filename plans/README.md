# Game Information System

A Spring Boot application for managing your video game collection. This system allows you to track games you own, including details like release dates, gaming platforms, ownership status, and backup information.

## Features

- **Complete CRUD Operations**: Create, read, update, and delete game information
- **Search Functionality**: Find games by name, system, ownership status, or backup status
- **RESTful API**: Well-designed API endpoints with proper HTTP methods
- **Data Validation**: Input validation to ensure data integrity
- **Error Handling**: Comprehensive error handling with appropriate HTTP status codes
- **API Documentation**: Swagger/OpenAPI documentation for easy API exploration
- **PostgreSQL Database**: Persistent storage using PostgreSQL running in Docker
- **Comprehensive Testing**: Unit and integration tests for all components

## Technology Stack

- **Backend**: Java 17, Spring Boot 4.x
- **Database**: PostgreSQL 16
- **Containerization**: Docker, Docker Compose
- **Documentation**: Swagger/OpenAPI
- **Testing**: JUnit 5, Mockito, Spring Test

## Project Structure

The project follows a standard layered architecture:

- **Entity Layer**: Database models
- **Repository Layer**: Data access interfaces
- **Service Layer**: Business logic
- **Controller Layer**: REST API endpoints
- **DTO Layer**: Data transfer objects for API communication
- **Exception Layer**: Custom exceptions and global error handling

## Getting Started

### Prerequisites

- Java 17 or higher
- Maven
- Docker and Docker Compose

### Setup Instructions

1. **Clone the repository**

```bash
git clone <repository-url>
cd game-info-system
```

2. **Start the PostgreSQL database**

```bash
docker-compose up -d
```

3. **Build the application**

```bash
./mvnw clean package -DskipTests
```

4. **Run the application**

```bash
./mvnw spring-boot:run
```

5. **Access the application**

- API: http://localhost:8080/api/games
- Swagger UI: http://localhost:8080/swagger-ui.html
- API Docs: http://localhost:8080/api-docs

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/games | Get all games |
| GET | /api/games/{id} | Get game by ID |
| POST | /api/games | Create new game |
| PUT | /api/games/{id} | Update game by ID |
| DELETE | /api/games/{id} | Delete game by ID |
| GET | /api/games/search | Search games by criteria |

## Sample API Usage

### Create a new game

```bash
curl -X POST http://localhost:8080/api/games \
  -H "Content-Type: application/json" \
  -d '{
    "name": "The Legend of Zelda: Breath of the Wild",
    "releaseDate": "2017-03-03",
    "system": "Nintendo Switch",
    "owned": true,
    "hasBackup": false
  }'
```

### Get all games

```bash
curl -X GET http://localhost:8080/api/games
```

### Search for games by system

```bash
curl -X GET "http://localhost:8080/api/games/search?system=Nintendo%20Switch"
```

## Development

### Running Tests

```bash
./mvnw test
```

### Building for Production

```bash
./mvnw clean package
```

## Project Documentation

For more detailed information, refer to the following documents:

- [Architecture Overview](plans/architecture.md)
- [Implementation Plan](plans/implementation_plan.md)
- [Docker Setup Guide](plans/docker_setup.md)
- [Testing Strategy](plans/testing_strategy.md)
- [Implementation Roadmap](plans/implementation_roadmap.md)

## License

This project is licensed under the MIT License - see the LICENSE file for details.