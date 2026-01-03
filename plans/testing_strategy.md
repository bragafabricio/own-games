# Testing Strategy for Game Information System

This document outlines the testing approach for the Game Information System to ensure quality and reliability.

## 1. Unit Testing

### Repository Layer Tests

Test the repository methods using an in-memory H2 database:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class GameInfoRepositoryTest {

    @Autowired
    private GameInfoRepository gameInfoRepository;
    
    @Test
    public void testFindBySystem() {
        // Given
        GameInfo game1 = new GameInfo();
        game1.setName("Game 1");
        game1.setSystem("Nintendo Switch");
        game1.setOwned(true);
        game1.setHasBackup(false);
        
        GameInfo game2 = new GameInfo();
        game2.setName("Game 2");
        game2.setSystem("PlayStation 5");
        game2.setOwned(true);
        game2.setHasBackup(true);
        
        gameInfoRepository.saveAll(List.of(game1, game2));
        
        // When
        List<GameInfo> result = gameInfoRepository.findBySystem("Nintendo Switch");
        
        // Then
        assertEquals(1, result.size());
        assertEquals("Game 1", result.get(0).getName());
    }
    
    // Additional tests for other repository methods
}
```

### Service Layer Tests

Test the service layer with mocked repository:

```java
@ExtendWith(MockitoExtension.class)
public class GameInfoServiceTest {

    @Mock
    private GameInfoRepository gameInfoRepository;
    
    @InjectMocks
    private GameInfoServiceImpl gameInfoService;
    
    @Test
    public void testGetGameById() {
        // Given
        Long id = 1L;
        GameInfo game = new GameInfo();
        game.setId(id);
        game.setName("Test Game");
        game.setSystem("Test System");
        game.setOwned(true);
        game.setHasBackup(false);
        
        when(gameInfoRepository.findById(id)).thenReturn(Optional.of(game));
        
        // When
        GameInfoResponseDto result = gameInfoService.getGameById(id);
        
        // Then
        assertNotNull(result);
        assertEquals("Test Game", result.getName());
        assertEquals("Test System", result.getSystem());
        assertTrue(result.isOwned());
        assertFalse(result.isHasBackup());
        
        verify(gameInfoRepository).findById(id);
    }
    
    @Test
    public void testGetGameByIdNotFound() {
        // Given
        Long id = 1L;
        when(gameInfoRepository.findById(id)).thenReturn(Optional.empty());
        
        // When & Then
        assertThrows(ResourceNotFoundException.class, () -> gameInfoService.getGameById(id));
        verify(gameInfoRepository).findById(id);
    }
    
    // Additional tests for other service methods
}
```

### Controller Layer Tests

Test the controller layer with mocked service:

```java
@WebMvcTest(GameInfoController.class)
public class GameInfoControllerTest {

    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private GameInfoService gameInfoService;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    public void testGetGameById() throws Exception {
        // Given
        Long id = 1L;
        GameInfoResponseDto gameDto = new GameInfoResponseDto();
        gameDto.setId(id);
        gameDto.setName("Test Game");
        gameDto.setSystem("Test System");
        gameDto.setOwned(true);
        gameDto.setHasBackup(false);
        
        when(gameInfoService.getGameById(id)).thenReturn(gameDto);
        
        // When & Then
        mockMvc.perform(get("/api/games/{id}", id))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(id))
                .andExpect(jsonPath("$.name").value("Test Game"))
                .andExpect(jsonPath("$.system").value("Test System"))
                .andExpect(jsonPath("$.owned").value(true))
                .andExpect(jsonPath("$.hasBackup").value(false));
        
        verify(gameInfoService).getGameById(id);
    }
    
    @Test
    public void testCreateGame() throws Exception {
        // Given
        GameInfoRequestDto requestDto = new GameInfoRequestDto();
        requestDto.setName("New Game");
        requestDto.setSystem("New System");
        requestDto.setOwned(true);
        requestDto.setHasBackup(false);
        
        GameInfoResponseDto responseDto = new GameInfoResponseDto();
        responseDto.setId(1L);
        responseDto.setName("New Game");
        responseDto.setSystem("New System");
        responseDto.setOwned(true);
        responseDto.setHasBackup(false);
        
        when(gameInfoService.createGame(any(GameInfoRequestDto.class))).thenReturn(responseDto);
        
        // When & Then
        mockMvc.perform(post("/api/games")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(requestDto)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1L))
                .andExpect(jsonPath("$.name").value("New Game"))
                .andExpect(jsonPath("$.system").value("New System"))
                .andExpect(jsonPath("$.owned").value(true))
                .andExpect(jsonPath("$.hasBackup").value(false));
        
        verify(gameInfoService).createGame(any(GameInfoRequestDto.class));
    }
    
    // Additional tests for other controller methods
}
```

## 2. Integration Testing

Create integration tests to verify the interaction between different layers:

```java
@SpringBootTest
@AutoConfigureTestDatabase
@AutoConfigureMockMvc
public class GameInfoIntegrationTest {

    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private GameInfoRepository gameInfoRepository;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @BeforeEach
    public void setup() {
        gameInfoRepository.deleteAll();
    }
    
    @Test
    public void testCreateAndGetGame() throws Exception {
        // Create a game
        GameInfoRequestDto requestDto = new GameInfoRequestDto();
        requestDto.setName("Integration Test Game");
        requestDto.setSystem("Test System");
        requestDto.setOwned(true);
        requestDto.setHasBackup(false);
        
        MvcResult createResult = mockMvc.perform(post("/api/games")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(requestDto)))
                .andExpect(status().isCreated())
                .andReturn();
        
        // Extract ID from response
        GameInfoResponseDto createdGame = objectMapper.readValue(
                createResult.getResponse().getContentAsString(),
                GameInfoResponseDto.class);
        
        // Get the game by ID
        mockMvc.perform(get("/api/games/{id}", createdGame.getId()))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("Integration Test Game"))
                .andExpect(jsonPath("$.system").value("Test System"))
                .andExpect(jsonPath("$.owned").value(true))
                .andExpect(jsonPath("$.hasBackup").value(false));
    }
    
    @Test
    public void testFullCrudFlow() throws Exception {
        // Create
        GameInfoRequestDto createDto = new GameInfoRequestDto();
        createDto.setName("CRUD Test Game");
        createDto.setSystem("CRUD System");
        createDto.setOwned(true);
        createDto.setHasBackup(false);
        
        MvcResult createResult = mockMvc.perform(post("/api/games")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(createDto)))
                .andExpect(status().isCreated())
                .andReturn();
        
        GameInfoResponseDto createdGame = objectMapper.readValue(
                createResult.getResponse().getContentAsString(),
                GameInfoResponseDto.class);
        Long gameId = createdGame.getId();
        
        // Read
        mockMvc.perform(get("/api/games/{id}", gameId))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("CRUD Test Game"));
        
        // Update
        GameInfoRequestDto updateDto = new GameInfoRequestDto();
        updateDto.setName("Updated CRUD Game");
        updateDto.setSystem("CRUD System");
        updateDto.setOwned(true);
        updateDto.setHasBackup(true);
        
        mockMvc.perform(put("/api/games/{id}", gameId)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(updateDto)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("Updated CRUD Game"))
                .andExpect(jsonPath("$.hasBackup").value(true));
        
        // Delete
        mockMvc.perform(delete("/api/games/{id}", gameId))
                .andExpect(status().isNoContent());
        
        // Verify deletion
        mockMvc.perform(get("/api/games/{id}", gameId))
                .andExpect(status().isNotFound());
    }
}
```

## 3. Test Configuration

Create a test configuration file `src/test/resources/application-test.properties`:

```properties
# Test Database Configuration
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect

# JPA/Hibernate Configuration for Tests
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
```

## 4. Test Dependencies

Add the following dependencies to your `pom.xml` for testing:

```xml
<!-- H2 Database for Testing -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>

<!-- Spring Boot Test -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

## 5. Test Coverage Goals

Aim for the following test coverage:
- Repository layer: 90%+ coverage
- Service layer: 85%+ coverage
- Controller layer: 80%+ coverage
- Overall application: 80%+ coverage

## 6. Testing Best Practices

1. **Isolation**: Each test should be independent and not rely on the state from other tests
2. **Readability**: Use descriptive test names that explain what is being tested
3. **Arrange-Act-Assert**: Structure tests with clear setup, action, and verification phases
4. **Test Edge Cases**: Include tests for error conditions and edge cases
5. **Clean Up**: Ensure tests clean up after themselves, especially when using external resources

## 7. Continuous Integration

Configure your CI pipeline to:
1. Run all tests on each commit
2. Generate test coverage reports
3. Fail the build if test coverage falls below thresholds
4. Fail the build if any tests fail

## 8. Manual Testing Checklist

Before releasing:
- [ ] Verify all CRUD operations through the API
- [ ] Test with valid and invalid input data
- [ ] Check error handling and response codes
- [ ] Verify database constraints are enforced
- [ ] Test search functionality with various parameters