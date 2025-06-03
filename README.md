# 🎙️ Intégration ElevenLabs TTS avec Spring Boot

Ce projet Spring Boot montre comment intégrer l’API **ElevenLabs** pour générer de la synthèse vocale (TTS) en temps réel.

---

## 📁 Structure du Projet

```text
src/main/java/com/bewize/tts/
├── config/
│   ├── WebConfig.java
│   └── ElevenLabsConfig.java
├── controller/
│   ├── TTSController.java
│   └── ContentController.java
├── model/
│   ├── Content.java
│   ├── ContentType.java
│   ├── Subject.java
│   └── dto/
│       ├── TTSRequest.java
│       └── TTSResponse.java
├── repository/
│   └── ContentRepository.java
├── service/
│   ├── TTSService.java
│   └── ContentService.java
└── exception/
    ├── GlobalExceptionHandler.java
    └── TTSServiceException.java

src/main/resources/
├── application.properties
└── data.sql
```

---

## 📌 Étape 1 : Entité `Content.java`

```java
@Entity
@Table(name = "content")
@Data
public class Content {

    @Id
    private String id;

    @Column(columnDefinition = "TEXT")
    private String text;

    @Enumerated(EnumType.STRING)
    private Subject subject;

    @Transient
    private byte[] audioStream;
}

```

---

## 📌 Étape 2 : Enum `Subject.java`


```java
public enum Subject {
    ENGLISH,
    FRENCH,
    MATH,
    ARABIC
}
```

---

## 📌 Étape 3 : DTOs `TTSRequest.java` et `TTSResponse.java`

```java
@Data
public class TTSRequest {
    private String text;
    private Subject subject;
}

```

```java
@Data
public class TTSResponse {
    private String contentId;
    private byte[] audioStream;
}

```

---

## 📌 Étape 4 : Configuration `ElevenLabsConfig.java`

```java
@Configuration
public class ElevenLabsConfig {

    @Value("${elevenlabs.api.key}")
    private String apiKey;

    @Value("${elevenlabs.api.url}")
    private String apiUrl;

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public String getApiKey() {
        return apiKey;
    }

    public String getApiUrl() {
        return apiUrl;
    }
}
```

---

## 📌 Étape 5 : Service `TTSService.java`

```java
@Service
public class TTSService {

    private final RestTemplate restTemplate;
    private final ElevenLabsConfig elevenLabsConfig;

    private static final Map<Subject, String> VOICE_IDS = Map.of(
        Subject.ENGLISH, "9BWtsMINqrJLrRacOk9x",
        Subject.FRENCH, "pFZP5JQG7iQjIQuC4Bku",
        Subject.MATH, "pFZP5JQG7iQjIQuC4Bku",
        Subject.ARABIC, "tavIIPLplRB883FzWU0V"
    );

    public TTSService(RestTemplate restTemplate, ElevenLabsConfig config) {
        this.restTemplate = restTemplate;
        this.elevenLabsConfig = config;
    }

    public byte[] generateTTS(String text, Subject subject) {
        String voiceId = VOICE_IDS.getOrDefault(subject, VOICE_IDS.get(Subject.FRENCH));
        String url = String.format("%s/v1/text-to-speech/%s", elevenLabsConfig.getApiUrl(), voiceId);

        HttpHeaders headers = new HttpHeaders();
        headers.set("xi-api-key", elevenLabsConfig.getApiKey());
        headers.setContentType(MediaType.APPLICATION_JSON);

        Map<String, Object> body = Map.of(
            "text", text,
            "model_id", "eleven_multilingual_v2",
            "voice_settings", Map.of("stability", 0.5, "similarity_boost", 0.75, "speed", 1.0)
        );

        HttpEntity<Map<String, Object>> request = new HttpEntity<>(body, headers);
        ResponseEntity<byte[]> response = restTemplate.exchange(url, HttpMethod.POST, request, byte[].class);

        if (response.getStatusCode().is2xxSuccessful() && response.getBody() != null) {
            return response.getBody();
        } else {
            throw new TTSServiceException("Erreur API: " + response.getStatusCode());
        }
    }

    public Content generateContentWithAudio(TTSRequest request) {
        Content content = new Content();
        content.setId(UUID.randomUUID().toString());
        content.setText(request.getText());
        content.setSubject(request.getSubject());
        content.setAudioStream(generateTTS(request.getText(), request.getSubject()));
        return content;
    }
}

```

---

## 📌 Étape 6 : Contrôleur `TTSController.java`

```java
@RestController
@RequestMapping("/api/tts")
public class TTSController {

    private final TTSService ttsService;

    public TTSController(TTSService service) {
        this.ttsService = service;
    }

    @PostMapping("/generate")
    public ResponseEntity<TTSResponse> generate(@RequestBody TTSRequest request) {
        Content content = ttsService.generateContentWithAudio(request);

        TTSResponse response = new TTSResponse();
        response.setContentId(content.getId());
        response.setAudioStream(content.getAudioStream());

        return ResponseEntity.ok(response);
    }

    @GetMapping("/stream")
    public ResponseEntity<byte[]> streamAudio(@RequestParam String text,
                                              @RequestParam Subject subject) {
        byte[] audio = ttsService.generateTTS(text, subject);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.parseMediaType("audio/mpeg"));
        headers.set("Content-Disposition", "inline; filename=\"tts.mp3\"");
        headers.setContentLength(audio.length);

        return ResponseEntity.ok().headers(headers).body(audio);
    }
}

```

---


 


## 📌 Étape 8 : `application.properties`

```properties
server.port=8080

spring.datasource.url=jdbc:postgresql://localhost:5432/dev
spring.datasource.username=dev_user
spring.datasource.password=dev_pass

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

elevenlabs.api.key=YOUR_API_KEY
elevenlabs.api.url=https://api.elevenlabs.io
```

---

## ✅ Test de l’API

```bash
curl -X POST http://localhost:8080/api/tts/generate      -H "Content-Type: application/json"      -d '{
  "text": "Bonjour à tous !",
  "subject": "FRENCH",
  "contentType": "FLASHCARD_QUESTION"
}'
```

---

## 🧪 Tests

### 📁 Structure des tests

```text
src/test/java/com/bewize/tts/
├── service/
│   └── TTSServiceTest.java
├── controller/
│   └── TTSControllerTest.java
└── integration/
    └── TTSIntegrationTest.java
```

### 1️⃣ Test unitaire `TTSServiceTest.java`

```java
@ExtendWith(MockitoExtension.class)
public class TTSServiceTest {

    @Mock
    private RestTemplate restTemplate;

    @Mock
    private ElevenLabsConfig config;

    private TTSService ttsService;

    @BeforeEach
    void setUp() {
        when(config.getApiKey()).thenReturn("fake-api-key");
        when(config.getApiUrl()).thenReturn("https://api.elevenlabs.io");
        ttsService = new TTSService(restTemplate, config);
    }

    @Test
    void testGenerateTTS_success() {
        byte[] fakeAudio = "audio".getBytes(StandardCharsets.UTF_8);
        ResponseEntity<byte[]> response = new ResponseEntity<>(fakeAudio, HttpStatus.OK);

        when(restTemplate.exchange(
                anyString(),
                eq(HttpMethod.POST),
                any(HttpEntity.class),
                eq(byte[].class)
        )).thenReturn(response);

        byte[] result = ttsService.generateTTS("Bonjour", Subject.FRENCH);
        assertArrayEquals(fakeAudio, result);
    }

    @Test
    void testGenerateTTS_failure() {
        ResponseEntity<byte[]> response = new ResponseEntity<>(null, HttpStatus.BAD_REQUEST);

        when(restTemplate.exchange(
                anyString(),
                eq(HttpMethod.POST),
                any(HttpEntity.class),
                eq(byte[].class)
        )).thenReturn(response);

        assertThrows(TTSServiceException.class, () ->
                ttsService.generateTTS("Bonjour", Subject.FRENCH));
    }
}
```

### 2️⃣ Test Web Layer `TTSControllerTest.java`

```java
@WebMvcTest(TTSController.class)
class TTSControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private TTSService ttsService;

    @Test
    void testGenerateEndpoint_returnsAudio() throws Exception {
        Content content = new Content();
        content.setId("123");
        content.setAudioStream("audio".getBytes());

        when(ttsService.generateContentWithAudio(any(TTSRequest.class))).thenReturn(content);

        String json = """
        {
            "text": "Bonjour à tous !",
            "subject": "FRENCH",
            "contentType": "FLASHCARD_QUESTION"
        }
        """;

        mockMvc.perform(post("/api/tts/generate")
                .contentType(MediaType.APPLICATION_JSON)
                .content(json))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.contentId").value("123"))
            .andExpect(jsonPath("$.audioStream").exists());
    }
}
```

### 3️⃣ Test d’intégration `TTSIntegrationTest.java`

> 👉 Optionnel : pour isoler l’API externe, tu peux utiliser **WireMock**.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class TTSIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void integrationTestGenerateTTS() throws Exception {
        mockMvc.perform(get("/api/tts/stream")
                .param("text", "Bonjour")
                .param("subject", "FRENCH"))
            .andExpect(status().isOk())
            .andExpect(content().contentType("audio/mpeg"));
    }
}
```

### 🛠️ Dépendances de test (`pom.xml`)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 👨‍💻 Auteur

Zineb kbyla 