# Game Information System Implementation Plan

This document provides a detailed step-by-step guide for implementing the Game Information System.

## 1. Update Project Dependencies

Update `pom.xml` to include the following dependencies:

```xml
<!-- PostgreSQL Driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Spring Data JPA (already included) -->

<!-- Spring Web (already included) -->

<!-- Lombok (already included) -->

<!-- Validation -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- Swagger/OpenAPI for API documentation -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

## 2. Set Up Docker and PostgreSQL Configuration

Create a `docker-compose.yml` file in the project root:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: games-postgres
    environment:
      POSTGRES_DB: gamesdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

## 3. Configure Application Properties

Update `src/main/resources/application.properties`:

```properties
# Application name
spring.application.name=game-info-service

# PostgreSQL Configuration
spring.datasource.url=jdbc:postgresql://localhost:5432/gamesdb
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA/Hibernate Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Server Configuration
server.port=8080

# OpenAPI Configuration
springdoc.api-docs.path=/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
```

## 4. Design and Implement Entity Model

Create `src/main/java/com/owngames/demo/model/GameInfo.java`:

```java
package com.owngames.demo.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "game_info")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class GameInfo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(name = "release_date")
    private LocalDate releaseDate;

    @Column(nullable = false)
    private String system;

    @Column(nullable = false)
    private boolean owned;

    @Column(name = "has_backup", nullable = false)
    private boolean hasBackup;

    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
}
```

## 5. Create DTOs

Create request and response DTOs:

`src/main/java/com/owngames/demo/dto/GameInfoRequestDto.java`:

```java
package com.owngames.demo.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

import java.time.LocalDate;

@Data
public class GameInfoRequestDto {
    
    @NotBlank(message = "Game name is required")
    private String name;
    
    private LocalDate releaseDate;
    
    @NotBlank(message = "System is required")
    private String system;
    
    @NotNull(message = "Owned status is required")
    private Boolean owned;
    
    @NotNull(message = "Backup status is required")
    private Boolean hasBackup;
}
```

`src/main/java/com/owngames/demo/dto/GameInfoResponseDto.java`:

```java
package com.owngames.demo.dto;

import lombok.Data;

import java.time.LocalDate;
import java.time.LocalDateTime;

@Data
public class GameInfoResponseDto {
    private Long id;
    private String name;
    private LocalDate releaseDate;
    private String system;
    private boolean owned;
    private boolean hasBackup;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

## 6. Implement Repository Layer

Create `src/main/java/com/owngames/demo/repository/GameInfoRepository.java`:

```java
package com.owngames.demo.repository;

import com.owngames.demo.model.GameInfo;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface GameInfoRepository extends JpaRepository<GameInfo, Long> {
    
    List<GameInfo> findBySystem(String system);
    
    List<GameInfo> findByOwned(boolean owned);
    
    List<GameInfo> findByHasBackup(boolean hasBackup);
    
    @Query("SELECT g FROM GameInfo g WHERE LOWER(g.name) LIKE LOWER(CONCAT('%', :name, '%'))")
    List<GameInfo> findByNameContainingIgnoreCase(@Param("name") String name);
}
```

## 7. Implement Service Layer

Create service interface and implementation:

`src/main/java/com/owngames/demo/service/GameInfoService.java`:

```java
package com.owngames.demo.service;

import com.owngames.demo.dto.GameInfoRequestDto;
import com.owngames.demo.dto.GameInfoResponseDto;

import java.util.List;

public interface GameInfoService {
    
    List<GameInfoResponseDto> getAllGames();
    
    GameInfoResponseDto getGameById(Long id);
    
    GameInfoResponseDto createGame(GameInfoRequestDto gameInfoDto);
    
    GameInfoResponseDto updateGame(Long id, GameInfoRequestDto gameInfoDto);
    
    void deleteGame(Long id);
    
    List<GameInfoResponseDto> searchGames(String name, String system, Boolean owned, Boolean hasBackup);
}
```

`src/main/java/com/owngames/demo/service/GameInfoServiceImpl.java`:

```java
package com.owngames.demo.service;

import com.owngames.demo.dto.GameInfoRequestDto;
import com.owngames.demo.dto.GameInfoResponseDto;
import com.owngames.demo.exception.ResourceNotFoundException;
import com.owngames.demo.model.GameInfo;
import com.owngames.demo.repository.GameInfoRepository;
import org.springframework.beans.BeanUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class GameInfoServiceImpl implements GameInfoService {

    private final GameInfoRepository gameInfoRepository;

    @Autowired
    public GameInfoServiceImpl(GameInfoRepository gameInfoRepository) {
        this.gameInfoRepository = gameInfoRepository;
    }

    @Override
    public List<GameInfoResponseDto> getAllGames() {
        return gameInfoRepository.findAll().stream()
                .map(this::convertToDto)
                .collect(Collectors.toList());
    }

    @Override
    public GameInfoResponseDto getGameById(Long id) {
        GameInfo gameInfo = gameInfoRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Game not found with id: " + id));
        return convertToDto(gameInfo);
    }

    @Override
    public GameInfoResponseDto createGame(GameInfoRequestDto gameInfoDto) {
        GameInfo gameInfo = convertToEntity(gameInfoDto);
        GameInfo savedGameInfo = gameInfoRepository.save(gameInfo);
        return convertToDto(savedGameInfo);
    }

    @Override
    public GameInfoResponseDto updateGame(Long id, GameInfoRequestDto gameInfoDto) {
        GameInfo existingGame = gameInfoRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Game not found with id: " + id));
        
        // Update fields
        existingGame.setName(gameInfoDto.getName());
        existingGame.setReleaseDate(gameInfoDto.getReleaseDate());
        existingGame.setSystem(gameInfoDto.getSystem());
        existingGame.setOwned(gameInfoDto.getOwned());
        existingGame.setHasBackup(gameInfoDto.getHasBackup());
        
        GameInfo updatedGame = gameInfoRepository.save(existingGame);
        return convertToDto(updatedGame);
    }

    @Override
    public void deleteGame(Long id) {
        if (!gameInfoRepository.existsById(id)) {
            throw new ResourceNotFoundException("Game not found with id: " + id);
        }
        gameInfoRepository.deleteById(id);
    }

    @Override
    public List<GameInfoResponseDto> searchGames(String name, String system, Boolean owned, Boolean hasBackup) {
        List<GameInfo> results;
        
        if (name != null && !name.isEmpty()) {
            results = gameInfoRepository.findByNameContainingIgnoreCase(name);
        } else if (system != null && !system.isEmpty()) {
            results = gameInfoRepository.findBySystem(system);
        } else if (owned != null) {
            results = gameInfoRepository.findByOwned(owned);
        } else if (hasBackup != null) {
            results = gameInfoRepository.findByHasBackup(hasBackup);
        } else {
            results = gameInfoRepository.findAll();
        }
        
        return results.stream()
                .map(this::convertToDto)
                .collect(Collectors.toList());
    }
    
    // Helper methods for DTO conversion
    private GameInfoResponseDto convertToDto(GameInfo gameInfo) {
        GameInfoResponseDto dto = new GameInfoResponseDto();
        BeanUtils.copyProperties(gameInfo, dto);
        return dto;
    }
    
    private GameInfo convertToEntity(GameInfoRequestDto dto) {
        GameInfo entity = new GameInfo();
        BeanUtils.copyProperties(dto, entity);
        return entity;
    }
}
```

## 8. Create Exception Handling

Create exception classes:

`src/main/java/com/owngames/demo/exception/ResourceNotFoundException.java`:

```java
package com.owngames.demo.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {
    
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

`src/main/java/com/owngames/demo/exception/GlobalExceptionHandler.java`:

```java
package com.owngames.demo.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(ResourceNotFoundException ex) {
        ErrorResponse errorResponse = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(),
                ex.getMessage(),
                LocalDateTime.now()
        );
        return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Object> handleValidationExceptions(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        
        ValidationErrorResponse errorResponse = new ValidationErrorResponse(
                HttpStatus.BAD_REQUEST.value(),
                "Validation failed",
                LocalDateTime.now(),
                errors
        );
        
        return new ResponseEntity<>(errorResponse, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(Exception ex) {
        ErrorResponse errorResponse = new ErrorResponse(
                HttpStatus.INTERNAL_SERVER_ERROR.value(),
                ex.getMessage(),
                LocalDateTime.now()
        );
        return new ResponseEntity<>(errorResponse, HttpStatus.INTERNAL_SERVER_ERROR);
    }
    
    // Error response classes
    static class ErrorResponse {
        private int status;
        private String message;
        private LocalDateTime timestamp;
        
        public ErrorResponse(int status, String message, LocalDateTime timestamp) {
            this.status = status;
            this.message = message;
            this.timestamp = timestamp;
        }
        
        // Getters and setters
        public int getStatus() { return status; }
        public void setStatus(int status) { this.status = status; }
        public String getMessage() { return message; }
        public void setMessage(String message) { this.message = message; }
        public LocalDateTime getTimestamp() { return timestamp; }
        public void setTimestamp(LocalDateTime timestamp) { this.timestamp = timestamp; }
    }
    
    static class ValidationErrorResponse extends ErrorResponse {
        private Map<String, String> errors;
        
        public ValidationErrorResponse(int status, String message, LocalDateTime timestamp, Map<String, String> errors) {
            super(status, message, timestamp);
            this.errors = errors;
        }
        
        // Getter and setter
        public Map<String, String> getErrors() { return errors; }
        public void setErrors(Map<String, String> errors) { this.errors = errors; }
    }
}
```

## 9. Implement Controller Layer

Create `src/main/java/com/owngames/demo/controller/GameInfoController.java`:

```java
package com.owngames.demo.controller;

import com.owngames.demo.dto.GameInfoRequestDto;
import com.owngames.demo.dto.GameInfoResponseDto;
import com.owngames.demo.service.GameInfoService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/games")
@Tag(name = "Game Information", description = "Game Information Management API")
public class GameInfoController {

    private final GameInfoService gameInfoService;

    @Autowired
    public GameInfoController(GameInfoService gameInfoService) {
        this.gameInfoService = gameInfoService;
    }

    @GetMapping
    @Operation(summary = "Get all games", description = "Retrieve a list of all games")
    public ResponseEntity<List<GameInfoResponseDto>> getAllGames() {
        List<GameInfoResponseDto> games = gameInfoService.getAllGames();
        return ResponseEntity.ok(games);
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get game by ID", description = "Retrieve a specific game by its ID")
    public ResponseEntity<GameInfoResponseDto> getGameById(@PathVariable Long id) {
        GameInfoResponseDto game = gameInfoService.getGameById(id);
        return ResponseEntity.ok(game);
    }

    @PostMapping
    @Operation(summary = "Create new game", description = "Add a new game to the collection")
    public ResponseEntity<GameInfoResponseDto> createGame(@Valid @RequestBody GameInfoRequestDto gameInfoDto) {
        GameInfoResponseDto createdGame = gameInfoService.createGame(gameInfoDto);
        return new ResponseEntity<>(createdGame, HttpStatus.CREATED);
    }

    @PutMapping("/{id}")
    @Operation(summary = "Update game", description = "Update an existing game's information")
    public ResponseEntity<GameInfoResponseDto> updateGame(
            @PathVariable Long id,
            @Valid @RequestBody GameInfoRequestDto gameInfoDto) {
        GameInfoResponseDto updatedGame = gameInfoService.updateGame(id, gameInfoDto);
        return ResponseEntity.ok(updatedGame);
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "Delete game", description = "Remove a game from the collection")
    public ResponseEntity<Void> deleteGame(@PathVariable Long id) {
        gameInfoService.deleteGame(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/search")
    @Operation(summary = "Search games", description = "Search games by name, system, owned status, or backup status")
    public ResponseEntity<List<GameInfoResponseDto>> searchGames(
            @RequestParam(required = false) String name,
            @RequestParam(required = false) String system,
            @RequestParam(required = false) Boolean owned,
            @RequestParam(required = false) Boolean hasBackup) {
        
        List<GameInfoResponseDto> games = gameInfoService.searchGames(name, system, owned, hasBackup);
        return ResponseEntity.ok(games);
    }
}
```

## 10. Testing

Create test classes for each layer:

1. Repository tests
2. Service tests
3. Controller tests
4. Integration tests

## 11. Running the Application

1. Start the PostgreSQL database:
   ```
   docker-compose up -d
   ```

2. Run the Spring Boot application:
   ```
   ./mvnw spring-boot:run
   ```

3. Access the API at `http://localhost:8080/api/games`
4. Access the Swagger UI at `http://localhost:8080/swagger-ui.html`

## 12. Sample API Usage

### Create a new game
```
POST /api/games
Content-Type: application/json

{
  "name": "The Legend of Zelda: Breath of the Wild",
  "releaseDate": "2017-03-03",
  "system": "Nintendo Switch",
  "owned": true,
  "hasBackup": false
}
```

### Get all games
```
GET /api/games
```

### Get game by ID
```
GET /api/games/1
```

### Update a game
```
PUT /api/games/1
Content-Type: application/json

{
  "name": "The Legend of Zelda: Breath of the Wild",
  "releaseDate": "2017-03-03",
  "system": "Nintendo Switch",
  "owned": true,
  "hasBackup": true
}
```

### Delete a game
```
DELETE /api/games/1
```

### Search games
```
GET /api/games/search?system=Nintendo%20Switch