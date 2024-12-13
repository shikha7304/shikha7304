Method: Post

API Definition: /v1/merchant/generate/otp

Access Type: Open API

Request Body:



{
  "requestType": String,
  "userId": String
}





        
      request Type need to be Enum and can hold the value - LOGIN, RESET_PASSWORD

        
      userId will be either userId or email or mobile.

        
      Both fields are mandatory


Success Response Body

{
  "data": [
    {
       "requestId": UUID,
       "otpDescription": "OTP successfully generated for user: " + userId,
       "otp": Six Digit random number,
    }
   ],
  "status":1,
  "count": 1,
  "total": 1
}



Failure Response Body

{
  "status":0,
  "error": [
     {
       "errorCode": String,
       "errorMessage": String
     }
   ]
}


Development Steps for OTP Implementation 
Step 1: Add Library - Apache Kafka
Step 2: Configure - Set up a Spring Bean to configure the Kafka properties. These properties include topics for send SMS and email. 
This ensures OTP meets the required specifications. 
Step 3: Create OTPController - Develop a REST Controller to expose the OTP API endpoints for generation and validation. 

Generates an OTP with expiration time, publish OTP to Kafka SMS/Email topic API so that it can be received, also store it in the data base. 

Step 4: Develop Kafka Producers - Create two component class for Kafka producer and create AVRO object to publish OTP.


        
      Generate OTP with expiry. 

        
      Save it in DB <Table>. 

        
      Publish it via Kafka producer. 

Step 5: Create DAO and Repository 


        
      
OTP Entity: Define a JPA entity to represent OTP data, including: 
ID 
OTP 
Channel (Email/SMS)
Expiration time 
Request Id 
Request Type 
Created At 


        
      
OTP Dao: To call the repository layer 


        
      
OTP Repository: Use a Spring Data JPA repository to persist and retrieve OTP information. 


Summary of Key Components 


        
      
Library: Kafka for publish generation. 

        
      
Configuration: Defines Kafka topic, SMS template, expiry duration. 

        
      
Controller: Exposes endpoints for OTP generation and validation. 

        
      
Service: Handles OTP logic, including expiration. 

        
      
Repository: Stores OTP data in the database for validation and expiration. 

Acceptance Criteria 


        
      OTP should generate dynamically on each request with six digits random number. 




      Step 1: Add Apache Kafka Library
Add the Kafka dependency in your pom.xml (for Maven):

<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>3.0.0</version>
</dependency>
Step 2: Kafka Configuration
Create a Kafka configuration class for producer settings:

@Configuration
@EnableKafka
public class KafkaConfig {

    @Value("${kafka.bootstrap.servers}")
    private String bootstrapServers;

    @Value("${kafka.topic.sms}")
    private String smsTopic;

    @Value("${kafka.topic.email}")
    private String emailTopic;

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return new DefaultKafkaProducerFactory<>(configProps);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    public String getSmsTopic() {
        return smsTopic;
    }

    public String getEmailTopic() {
        return emailTopic;
    }
}
Step 3: OTP Enum
Define the request types as an Enum:

public enum RequestType {
    LOGIN,
    RESET_PASSWORD
}
Step 4: Create OTP Entity
Define the JPA entity for OTP management:

@Entity
@Table(name = "OTP_MANAGEMENT")
public class OtpEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private UUID id;

    @Enumerated(EnumType.STRING)
    private RequestType requestType;

    @Column(nullable = false)
    private String requestId;

    @Column(nullable = false)
    private String userId;

    @Column(nullable = false, length = 6)
    private String otpCode;

    @Column(nullable = false)
    private LocalDateTime expiryTime;

    @Column(nullable = false)
    private boolean isVerified;

    @Column(nullable = false)
    private LocalDateTime createdDate;

    @Column(nullable = false)
    private LocalDateTime updateDate;

    // Getters and setters
}
Step 5: Create Repository
Define the repository to interact with the database:

@Repository
public interface OtpRepository extends JpaRepository<OtpEntity, UUID> {
    Optional<OtpEntity> findByRequestIdAndUserId(String requestId, String userId);
}
Step 6: OTP Service
Implement the business logic in the service layer:

@Service
public class OtpService {

    @Autowired
    private OtpRepository otpRepository;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private KafkaConfig kafkaConfig;

    public OtpEntity generateOtp(RequestType requestType, String userId) {
        String otp = String.valueOf(new Random().nextInt(900000) + 100000); // Generate 6-digit OTP
        LocalDateTime expiryTime = LocalDateTime.now().plusMinutes(5);

        OtpEntity otpEntity = new OtpEntity();
        otpEntity.setRequestType(requestType);
        otpEntity.setRequestId(UUID.randomUUID().toString());
        otpEntity.setUserId(userId);
        otpEntity.setOtpCode(otp);
        otpEntity.setExpiryTime(expiryTime);
        otpEntity.setVerified(false);
        otpEntity.setCreatedDate(LocalDateTime.now());
        otpEntity.setUpdateDate(LocalDateTime.now());

        // Save to DB
        otpRepository.save(otpEntity);

        // Publish OTP to Kafka
        String kafkaMessage = String.format("OTP: %s for user: %s", otp, userId);
        kafkaTemplate.send(kafkaConfig.getSmsTopic(), kafkaMessage);

        return otpEntity;
    }
}
Step 7: Create OTP Controller
Develop the REST controller to expose the OTP generation endpoint:

@RestController
@RequestMapping("/v1/merchant/generate")
public class OtpController {

    @Autowired
    private OtpService otpService;

    @PostMapping("/otp")
    public ResponseEntity<?> generateOtp(@RequestBody @Valid OtpRequest request) {
        try {
            OtpEntity otp = otpService.generateOtp(RequestType.valueOf(request.getRequestType()), request.getUserId());

            Map<String, Object> response = new HashMap<>();
            response.put("data", List.of(Map.of(
                    "requestId", otp.getRequestId(),
                    "otpDescription", "OTP successfully generated for user: " + request.getUserId(),
                    "otp", otp.getOtpCode()
            )));
            response.put("status", 1);
            response.put("count", 1);
            response.put("total", 1);

            return ResponseEntity.ok(response);
        } catch (IllegalArgumentException e) {
            return ResponseEntity.badRequest().body(Map.of(
                    "status", 0,
                    "error", List.of(Map.of(
                            "errorCode", "INVALID_REQUEST_TYPE",
                            "errorMessage", "Invalid request type. Allowed values: LOGIN, RESET_PASSWORD"
                    ))
            ));
        }
    }
}
Step 8: Create Request DTO
Define the request payload and add validation:

public class OtpRequest {

    @NotNull(message = "Request type is mandatory")
    private String requestType;

    @NotNull(message = "User ID is mandatory")
    private String userId;

    // Getters and setters
}
Step 9: Acceptance Criteria
OTP Generation:
Generates a 6-digit random OTP.
OTP expires in 5 minutes.
Database Storage:
Stores OTP with relevant details in the database.
Kafka Publishing:
Publishes OTP messages to Kafka topics for SMS and email.
Validation:
Validates requestType as LOGIN or RESET_PASSWORD.
Ensures userId is not null or empty.
Logging:
Logs all requests and responses for traceability.


        
      OTP should expire after 5 minutes. 

        
      Validation should confirm the user-provided OTP matches the stored OTP. 

        
      Users can resend the OTP if needed.

        
      All requests and responses must be logged for traceability. 
