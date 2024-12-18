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



 if (otpGenerationRequest.getUserId().contains(String.valueOf('@'))) {
            EmailDTO emailDTO = new EmailDTO();
            isOtpSend = emailService.sendEmail(emailDTO);

        } else {
            try {
                SmsDTO smsDTO = new SmsDTO();
                isOtpSend = smsService.sendSMS(smsDTO);

            }
            catch (Exception e) {
                throw new RuntimeException(e);
            }
        }

import java.util.regex.Pattern;

public class RegexValidation {

    // Regex for validating phone number
    private static final String PHONE_REGEX = "^\\+?[0-9]{10,15}$";

    // Regex for validating email
    private static final String EMAIL_REGEX = "^[A-Za-z0-9+_.-]+@(.+)$";

    // Validate phone number
    public static boolean validatePhone(String phone) {
        return phone != null && Pattern.matches(PHONE_REGEX, phone);
    }

    // Validate email
    public static boolean validateEmail(String email) {
        return email != null && Pattern.matches(EMAIL_REGEX, email);
    }

    // Validate user ID (alphanumeric check)
    public static boolean validateUserId(String userId) {
        return userId != null && userId.matches("^[a-zA-Z0-9]+$");
    }
}



ou said:
E CREATE TABLE OTE
_MANAGEMENT (
ID
RAW (16) PRIMARY KEY NOT NULL,
REQUEST_ TYPE
VARCHARZ (255) ,
--Unique identifier for each OTP entry.
REQUEST_ ID
RAW (16) NOT NULL,
-Type of request associated with the OTP, e.g., login, resetPassword.
USER ED
RAW (16) NOT NULL,
--A unique identifier for the specific request (e.g., session ID or transa
OTE_CODE
VARCHAR (10) 4
-Reference to the user requesting the OTP.
=-the generated OTP.
EXPIRY TIME
NUMBER,
IS_VERIFIED
--Timestamp indicating when the OTE expires.
BOOLEAN,
CREATED DATE
-Flag indicating whether the OTP has been successfully verified.
NUMBER,
--Timestamp of when the OTP was generated.
UPDATE_DATE
NUMBER,
CHANNEL
-Timestamp of the last update (e.g., verification status change) .
VARCHAR2 (100)
) =
I
Script Output X
目
UPDATE_ DATE
CHANNEL
Error report -
ORA-00902: invalid datatype
00902. 00000 - invalid datatype*
‹Cause:
*Action:
ChatGPT said:

ChatGPT
The error ORA-00902: invalid datatype occurs because Oracle Database does not recognize one or more of the data types used in the CREATE TABLE statement. Let's fix the issues:

Invalid data types:
RAW type is supported, but ensure its usage matches your intention.
VARCHARZ is invalid; replace it with VARCHAR2.
BOOLEAN is not a valid Oracle datatype; replace it with NUMBER(1) (0 for false, 1 for true) or a CHAR(1) field ('Y'/'N').
Missing constraints and defaults: Add appropriate constraints or defaults where needed.
Here’s the corrected script:

CREATE TABLE OTP_MANAGEMENT (
    ID RAW(16) PRIMARY KEY NOT NULL, -- Unique identifier for each OTP entry.
    REQUEST_TYPE VARCHAR2(255), -- Type of request associated with the OTP, e.g., login, resetPassword.
    REQUEST_ID RAW(16) NOT NULL, -- A unique identifier for the specific request (e.g., session ID or transaction ID).
    USER_ID RAW(16) NOT NULL, -- Reference to the user requesting the OTP.
    OTP_CODE VARCHAR2(10), -- The generated OTP.
    EXPIRY_TIME NUMBER, -- Timestamp indicating when the OTP expires.
    IS_VERIFIED NUMBER(1) DEFAULT 0, -- Flag indicating whether the OTP has been successfully verified (0 = false, 1 = true).
    CREATED_DATE DATE, -- Timestamp of when the OTP was generated.
    UPDATE_DATE DATE, -- Timestamp of the last update (e.g., verification status change).
    CHANNEL VARCHAR2(100) -- Channel through which the OTP was sent.
);
Explanation of Fixes
VARCHARZ replaced with VARCHAR2: Oracle uses VARCHAR2 for variable-length string data.
BOOLEAN replaced with NUMBER(1): Oracle does not directly support BOOLEAN. Use NUMBER(1) with values like 0 (false) and 1 (true).
DATE for timestamps: Use DATE for all date and time fields instead of NUMBER.
Let me know if you need further adjustments or explanations!





You said:
I need to call utility class to send otp to email and sms based on the channel from the otp service class and i need to save the data in the notification table in value column along with some message so how can i do that ?
ChatGPT said:

ChatGPT
To achieve this, you can follow these steps:

1. Utility Class to Send OTP
Create or use an existing utility class to send the OTP via email or SMS based on the provided channel. For example:

Example Utility Class

public class NotificationUtil {

    public static void sendOtp(String channel, String recipient, String otp) {
        if ("EMAIL".equalsIgnoreCase(channel)) {
            sendEmail(recipient, otp);
        } else if ("SMS".equalsIgnoreCase(channel)) {
            sendSms(recipient, otp);
        } else {
            throw new IllegalArgumentException("Unsupported channel: " + channel);
        }
    }

    private static void sendEmail(String recipient, String otp) {
        // Logic to send OTP via email
        System.out.println("Sending OTP via Email to " + recipient + ": " + otp);
    }

    private static void sendSms(String recipient, String otp) {
        // Logic to send OTP via SMS
        System.out.println("Sending OTP via SMS to " + recipient + ": " + otp);
    }
}
2. Service Class to Handle OTP Logic
Use the utility class in your service class to:

Generate the OTP.
Send it via the specified channel (email/SMS).
Save the notification data in the database.
Example Service Class

import java.util.HashMap;
import java.util.Map;

public class OtpService {

    public void generateAndSendOtp(String userId, String channel, String recipient) {
        // Step 1: Generate OTP
        String otp = generateOtp();

        // Step 2: Send OTP
        NotificationUtil.sendOtp(channel, recipient, otp);

        // Step 3: Save notification data in the notification table
        saveNotification(userId, channel, otp);
    }

    private String generateOtp() {
        // Example OTP generation logic
        int otp = (int) (Math.random() * 9000) + 1000; // 4-digit OTP
        return String.valueOf(otp);
    }

    private void saveNotification(String userId, String channel, String otp) {
        // Create a map to store the value column data
        Map<String, Object> valueColumnData = new HashMap<>();
        valueColumnData.put("channel", channel);
        valueColumnData.put("otp", otp);

        String message = "OTP sent successfully via " + channel;

        // Save data in the Notification table
        Notification notification = new Notification();
        notification.setUserId(userId);
        notification.setValue(valueColumnData.toString());
        notification.setMessage(message);

        // Call DAO/Repository to save the notification
        NotificationDao.save(notification);
    }
}
3. Notification Table Structure
The table NOTIFICATION could look like this:

CREATE TABLE NOTIFICATION (
    ID RAW(16) PRIMARY KEY NOT NULL,
    USER_ID RAW(16) NOT NULL,
    VALUE CLOB, -- Store JSON or stringified map
    MESSAGE VARCHAR2(255),
    CREATED_DATE DATE DEFAULT SYSDATE
);
4. Notification Entity Class
public class Notification {
    private String id;
    private String userId;
    private String value; // Store as JSON string or key-value pairs
    private String message;
    private Date createdDate;

    // Getters and Setters
}
5. Saving Notification Data
Your DAO/Repository class can handle database interaction:

public class NotificationDao {

    public static void save(Notification notification) {
        // Example logic to save notification to the database
        System.out.println("Saving notification: " + notification);
        // Use JDBC, JPA, or Hibernate to persist the data
    }
}
Example Workflow in the Service
Call the generateAndSendOtp method in your service:

public class Main {
    public static void main(String[] args) {
        OtpService otpService = new OtpService();

        // Example usage: Send OTP via email
        otpService.generateAndSendOtp("USER123", "EMAIL", "user@example.com");

        // Example usage: Send OTP via SMS
        otpService.generateAndSendOtp("USER123", "SMS", "1234567890");
    }
}
Output Example
OTP is sent via the specified channel:
Email: Sending OTP via Email to user@example.com: 1234
SMS: Sending OTP via SMS to 1234567890: 1234
Notification is saved in the database with:
Value: {"channel":"EMAIL","otp":"1234"}
Message: OTP sent successfully via EMAIL
Let me know if you need help customizing this further!





You said:
Getting error 


if
(isotpSend) {
OtpGenerationResponse otpGenerationResponse = new OtpGenerationResponse(otp.getUserId(), otpDescription: "OTP
successfully generated
return ResponseDto. <OtpGenerationResponse>builder(
• status (AppConstants. RESPONSE_SUCCESS)
•data (List.of(otpGenerationResponse))
• count (1L)
• total(1L)
•build();
2
3
34
35
86
87
88
89
90
91
return ResponseDto. <OtpGenerationResponse>builder(
•status(AppConstants.RESPONSE_FAILURE)
• data (List.of(sorpsend))
. count (1L)
.total (1L)
•build;
Required type:
List <OtpGenerationResponse>
Provided:
List <Boolean>
no instance(s) of type variables) exist so that Boolean conforms to OtpGenerationResponse inference variable E has incompatible bounds: equality constraints: OtpGenerationResponse lower bounds: Boolean
Remove redundant arguments to call 'of" Alt+Shift +Enter
More actions... Ctrl+1 Alt+Enter
© com. epay merchant.service.OtpService
private boolean isotpSend = false
ChatGPT said:

ChatGPT
The error occurs because you're trying to pass a List<Boolean> (specifically List.of(isotpSend)) where a List<OtpGenerationResponse> is expected. The issue lies in the mismatch between the types in your ResponseDto builder.

Problem Area
Your code snippet suggests that when isOtpSend is false, you're attempting to build a response with a List<Boolean>:

return ResponseDto.<OtpGenerationResponse>builder()
    .status(AppConstants.RESPONSE_FAILURE)
    .data(List.of(isOtpSend)) // This is incorrect
    .count(1L)
    .total(1L)
    .build();
Here, List.of(isOtpSend) creates a List<Boolean> because isOtpSend is a boolean. However, the ResponseDto is configured to expect List<OtpGenerationResponse> in the data field.

Solution
You need to build a proper OtpGenerationResponse object or provide a meaningful error response instead of directly passing the isOtpSend value. Here's the corrected code:

if (isOtpSend) {
    OtpGenerationResponse otpGenerationResponse = new OtpGenerationResponse(
        otp.getUserId(),
        "OTP successfully generated"
    );
    return ResponseDto.<OtpGenerationResponse>builder()
        .status(AppConstants.RESPONSE_SUCCESS)
        .data(List.of(otpGenerationResponse))
        .count(1L)
        .total(1L)
        .build();
} else {
    OtpGenerationResponse errorResponse = new OtpGenerationResponse(
        null,
        "Failed to send OTP"
    );
    return ResponseDto.<OtpGenerationResponse>builder()
        .status(AppConstants.RESPONSE_FAILURE)
        .data(List.of(errorResponse))
        .count(1L)
        .total(1L)
        .build();
}
Changes Made
Corrected Type in Failure Case:
Instead of passing a boolean (isOtpSend) to List.of(...), I created an appropriate OtpGenerationResponse object to represent the error.
This ensures that the data field always contains a List<OtpGenerationResponse>.
Added Meaningful Error Message:
When isOtpSend is false, the response now includes a descriptive error message ("Failed to send OTP").
Explanation of Error
The Java compiler enforces strict type checking. Since the ResponseDto builder is parameterized with <OtpGenerationResponse>, all elements in the data list must be of type OtpGenerationResponse. When you passed a boolean (isOtpSend), it violated the type contract, leading to the incompatible bounds error.

General Notes
Ensure that all elements in the data list match the expected type (OtpGenerationResponse).
Use meaningful messages and objects for both success and failure cases to make debugging easier.
Consider logging errors or exceptions when the OTP sending process fails for better traceability.
Let me know if you have further questions!





You said:
Otp generation for login reset password and change password how can we do that 
ChatGPT said:

ChatGPT
To implement OTP generation for functionalities such as login, reset password, and change password, follow these steps:

Step 1: Define OTP Table
Ensure your database table OTP_MANAGEMENT supports these operations. Below is the structure:

CREATE TABLE OTP_MANAGEMENT (
    ID RAW(16) PRIMARY KEY NOT NULL, -- Unique OTP entry ID
    USER_ID RAW(16) NOT NULL, -- User identifier
    REQUEST_TYPE VARCHAR2(50) NOT NULL, -- Type of request: login, resetPassword, changePassword
    OTP_CODE VARCHAR2(10) NOT NULL, -- Generated OTP
    EXPIRY_TIME DATE NOT NULL, -- Expiry timestamp for the OTP
    IS_VERIFIED NUMBER(1) DEFAULT 0, -- OTP verification status: 0 = Not Verified, 1 = Verified
    CREATED_DATE DATE DEFAULT SYSDATE, -- Timestamp of OTP creation
    CHANNEL VARCHAR2(50) NOT NULL -- Channel: email or SMS
);
Step 2: OTP Generation Logic
The OTP generation logic should:

Generate a random OTP.
Save the OTP details in the database.
Send the OTP to the user via the specified channel.
Example Code for OTP Service

import java.util.Date;
import java.util.Random;

public class OtpService {

    private static final int OTP_EXPIRY_MINUTES = 5;

    public String generateOtp(String userId, String requestType, String channel, String recipient) {
        // Step 1: Generate a random 6-digit OTP
        String otp = String.format("%06d", new Random().nextInt(999999));

        // Step 2: Calculate the expiry time
        Date expiryTime = new Date(System.currentTimeMillis() + OTP_EXPIRY_MINUTES * 60 * 1000);

        // Step 3: Save OTP details in the database
        saveOtpToDatabase(userId, requestType, otp, expiryTime, channel);

        // Step 4: Send OTP to the user
        NotificationUtil.sendOtp(channel, recipient, otp);

        return "OTP successfully generated and sent via " + channel;
    }

    private void saveOtpToDatabase(String userId, String requestType, String otp, Date expiryTime, String channel) {
        // Logic to save OTP details in the database
        Otp otpEntity = new Otp();
        otpEntity.setUserId(userId);
        otpEntity.setRequestType(requestType);
        otpEntity.setOtpCode(otp);
        otpEntity.setExpiryTime(expiryTime);
        otpEntity.setChannel(channel);
        otpEntity.setIsVerified(false);

        OtpRepository.save(otpEntity); // Save using DAO/Repository
    }
}
Step 3: OTP Verification Logic
Implement a method to verify the OTP entered by the user.

OTP Verification Code

public boolean verifyOtp(String userId, String requestType, String otp) {
    // Step 1: Retrieve the OTP entry from the database
    Otp otpEntity = OtpRepository.findByUserIdAndRequestType(userId, requestType);

    if (otpEntity == null) {
        throw new IllegalArgumentException("No OTP found for the given request.");
    }

    // Step 2: Check if the OTP is expired
    if (otpEntity.getExpiryTime().before(new Date())) {
        throw new IllegalStateException("OTP has expired.");
    }

    // Step 3: Check if the OTP matches and is not already verified
    if (!otpEntity.getOtpCode().equals(otp) || otpEntity.getIsVerified()) {
        return false;
    }

    // Step 4: Mark the OTP as verified and update the database
    otpEntity.setIsVerified(true);
    OtpRepository.update(otpEntity);

    return true;
}
Step 4: Controller Endpoints
Define endpoints for different operations like login, reset password, and change password.

Example Endpoints

@RestController
@RequestMapping("/api/otp")
public class OtpController {

    @Autowired
    private OtpService otpService;

    @PostMapping("/generate")
    public ResponseEntity<String> generateOtp(@RequestBody OtpRequest request) {
        String message = otpService.generateOtp(
            request.getUserId(),
            request.getRequestType(),
            request.getChannel(),
            request.getRecipient()
        );
        return ResponseEntity.ok(message);
    }

    @PostMapping("/verify")
    public ResponseEntity<String> verifyOtp(@RequestBody OtpVerifyRequest request) {
        boolean isValid = otpService.verifyOtp(
            request.getUserId(),
            request.getRequestType(),
            request.getOtp()
        );
        return isValid ? ResponseEntity.ok("OTP verified successfully.")
                       : ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Invalid OTP.");
    }
}
Step 5: Request DTOs
Define request objects for OTP generation and verification.

DTO for OTP Generation

public class OtpRequest {
    private String userId;
    private String requestType; // login, resetPassword, changePassword
    private String channel; // EMAIL, SMS
    private String recipient; // Email or phone number

    // Getters and Setters
}
DTO for OTP Verification

public class OtpVerifyRequest {
    private String userId;
    private String requestType; // login, resetPassword, changePassword
    private String otp;

    // Getters and Setters
}
Step 6: Notification Utility
Send OTP to the user based on the channel.

Example Notification Utility

public class NotificationUtil {

    public static void sendOtp(String channel, String recipient, String otp) {
        if ("EMAIL".equalsIgnoreCase(channel)) {
            sendEmail(recipient, otp);
        } else if ("SMS".equalsIgnoreCase(channel)) {
            sendSms(recipient, otp);
        } else {
            throw new IllegalArgumentException("Unsupported channel: " + channel);
        }
    }

    private static void sendEmail(String recipient, String otp) {
        // Logic to send OTP via email
        System.out.println("Sending OTP via Email to " + recipient + ": " + otp);
    }

    private static void sendSms(String recipient, String otp) {
        // Logic to send OTP via SMS
        System.out.println("Sending OTP via SMS to " + recipient + ": " + otp);
    }
}
Workflow Summary:
Login, Reset Password, Change Password:
User requests an OTP for one of these actions by calling /api/otp/generate.
OTP is sent via the specified channel and saved in the database.
OTP Verification:
User submits the OTP via /api/otp/verify.
OTP is verified, and appropriate actions (e.g., login, reset password) proceed on success.
Let me know if you need further assistance!





You said:
How to call the method of validator class from controller 
ChatGPT said:

ChatGPT
To call a method from a validator class in a controller, follow these steps:

1. Create the Validator Class
Ensure your validator class has the necessary validation logic as public methods.

Example Validator Class

import org.springframework.stereotype.Component;

@Component
public class OtpValidator {

    public void validateOtpRequest(String requestType, String channel) {
        if (requestType == null || requestType.isEmpty()) {
            throw new IllegalArgumentException("Request type cannot be null or empty");
        }
        if (!"EMAIL".equalsIgnoreCase(channel) && !"SMS".equalsIgnoreCase(channel)) {
            throw new IllegalArgumentException("Invalid channel. Only EMAIL or SMS are allowed.");
        }
    }
}
2. Inject the Validator Class into the Controller
Use @Autowired or constructor injection to use the validator class in the controller.

Example Controller

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/otp")
public class OtpController {

    private final OtpValidator otpValidator;
    private final OtpService otpService;

    @Autowired
    public OtpController(OtpValidator otpValidator, OtpService otpService) {
        this.otpValidator = otpValidator;
        this.otpService = otpService;
    }

    @PostMapping("/generate")
    public ResponseEntity<String> generateOtp(@RequestBody OtpRequest request) {
        // Step 1: Validate the request using the validator class
        otpValidator.validateOtpRequest(request.getRequestType(), request.getChannel());

        // Step 2: Process OTP generation
        String message = otpService.generateOtp(
            request.getUserId(),
            request.getRequestType(),
            request.getChannel(),
            request.getRecipient()
        );

        return ResponseEntity.ok(message);
    }
}
3. Request DTO
Define a DTO for the incoming request to pass data to the controller.

public class OtpRequest {
    private String userId;
    private String requestType; // e.g., login, resetPassword, changePassword
    private String channel; // EMAIL or SMS
    private String recipient; // Email or phone number

    // Getters and Setters
}
4. Workflow in the Controller
Inject Validator Class: Use @Autowired to inject the OtpValidator into your controller.
Call the Validator Method: Call the validator method (validateOtpRequest) at the beginning of your controller method to validate the inputs.
Handle Validation Exceptions: Ensure that exceptions thrown by the validator are handled appropriately.
5. Exception Handling (Optional)
If you want to handle exceptions globally, you can use a @ControllerAdvice class.

Example Global Exception Handler

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleIllegalArgumentException(IllegalArgumentException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(ex.getMessage());
    }
}
6. Example Request Validation Flow
Incoming Request:
{
    "userId": "123",
    "requestType": "login",
    "channel": "EMAIL",
    "recipient": "user@example.com"
}
Validator Logic:
Validates that requestType is not null or empty.
Ensures channel is either EMAIL or SMS.
Controller Workflow:
Calls OtpValidator.validateOtpRequest(requestType, channel).
If validation passes, calls OtpService.generateOtp.
If validation fails, throws an exception and returns an appropriate error response.
This approach keeps validation logic separate from business logic, making your code more modular and maintainable. Let me know if you need further help!





You said:
How to implement these two in my classes 
ChatGPT said:

ChatGPT
To integrate the validator class and OTP generation logic into your existing classes, here is the detailed implementation step-by-step:

1. OTP Validator Class

This class ensures that incoming requests are valid before proceeding to the business logic.

Create OtpValidator
import org.springframework.stereotype.Component;

@Component
public class OtpValidator {

    // Method to validate request type and channel
    public void validateOtpRequest(String requestType, String channel) {
        // Validate requestType
        if (requestType == null || requestType.isEmpty()) {
            throw new IllegalArgumentException("Request type cannot be null or empty.");
        }

        // Validate supported channels
        if (!"EMAIL".equalsIgnoreCase(channel) && !"SMS".equalsIgnoreCase(channel)) {
            throw new IllegalArgumentException("Invalid channel. Supported channels are EMAIL or SMS.");
        }
    }
}
2. OTP Service Class

This class will:

Generate the OTP.
Save it to the database.
Send the OTP to the user.
Update OtpService
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.util.Date;
import java.util.Random;

@Service
public class OtpService {

    private static final int OTP_EXPIRY_MINUTES = 5;

    @Autowired
    private NotificationUtil notificationUtil;

    @Autowired
    private OtpRepository otpRepository; // Your DAO or Repository to interact with OTP_MANAGEMENT table.

    public String generateOtp(String userId, String requestType, String channel, String recipient) {
        // Step 1: Generate a random 6-digit OTP
        String otpCode = generateRandomOtp();

        // Step 2: Calculate expiry time (5 minutes from now)
        Date expiryTime = new Date(System.currentTimeMillis() + OTP_EXPIRY_MINUTES * 60 * 1000);

        // Step 3: Save OTP details into the database
        saveOtpToDatabase(userId, requestType, otpCode, expiryTime, channel);

        // Step 4: Send the OTP to the recipient via EMAIL or SMS
        notificationUtil.sendOtp(channel, recipient, otpCode);

        return "OTP successfully generated and sent via " + channel;
    }

    private String generateRandomOtp() {
        return String.format("%06d", new Random().nextInt(999999));
    }

    private void saveOtpToDatabase(String userId, String requestType, String otpCode, Date expiryTime, String channel) {
        OtpEntity otpEntity = new OtpEntity();
        otpEntity.setUserId(userId);
        otpEntity.setRequestType(requestType);
        otpEntity.setOtpCode(otpCode);
        otpEntity.setExpiryTime(expiryTime);
        otpEntity.setIsVerified(false);
        otpEntity.setChannel(channel);
        otpEntity.setCreatedDate(new Date());

        otpRepository.save(otpEntity); // Save to the database using repository
    }
}
3. Notification Utility Class

This is used to send OTP to either Email or SMS.

Update NotificationUtil
import org.springframework.stereotype.Component;

@Component
public class NotificationUtil {

    public void sendOtp(String channel, String recipient, String otpCode) {
        if ("EMAIL".equalsIgnoreCase(channel)) {
            sendEmail(recipient, otpCode);
        } else if ("SMS".equalsIgnoreCase(channel)) {
            sendSms(recipient, otpCode);
        } else {
            throw new IllegalArgumentException("Unsupported channel: " + channel);
        }
    }

    private void sendEmail(String recipient, String otpCode) {
        // Mock email logic - replace with actual email sender library
        System.out.println("Sending OTP [" + otpCode + "] to EMAIL: " + recipient);
    }

    private void sendSms(String recipient, String otpCode) {
        // Mock SMS logic - replace with actual SMS sender library
        System.out.println("Sending OTP [" + otpCode + "] to SMS: " + recipient);
    }
}
4. OTP Controller Class

Integrate the OtpValidator and OtpService into your controller.

Update OtpController
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/otp")
public class OtpController {

    @Autowired
    private OtpValidator otpValidator;

    @Autowired
    private OtpService otpService;

    @PostMapping("/generate")
    public ResponseEntity<String> generateOtp(@RequestBody OtpRequest request) {
        // Step 1: Validate the incoming request
        otpValidator.validateOtpRequest(request.getRequestType(), request.getChannel());

        // Step 2: Call the OTP Service to generate and send OTP
        String responseMessage = otpService.generateOtp(
            request.getUserId(),
            request.getRequestType(),
            request.getChannel(),
            request.getRecipient()
        );

        return ResponseEntity.ok(responseMessage);
    }
}
5. Request DTO Class

Create a DTO class to capture the incoming request body.

Create OtpRequest
public class OtpRequest {
    private String userId;       // User ID
    private String requestType;  // e.g., login, resetPassword, changePassword
    private String channel;      // EMAIL or SMS
    private String recipient;    // Email address or phone number

    // Getters and Setters
}
6. OTP Entity Class

Define the entity representing the OTP_MANAGEMENT table.

Create OtpEntity
import javax.persistence.*;
import java.util.Date;

@Entity
@Table(name = "OTP_MANAGEMENT")
public class OtpEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private String id;

    private String userId;
    private String requestType;
    private String otpCode;
    private Date expiryTime;
    private Boolean isVerified;
    private String channel;
    private Date createdDate;

    // Getters and Setters
}
7. OTP Repository

Create a repository interface for database operations.

Create OtpRepository
import org.springframework.data.jpa.repository.JpaRepository;

public interface OtpRepository extends JpaRepository<OtpEntity, String> {
    OtpEntity findByUserIdAndRequestType(String userId, String requestType);
}
Workflow Summary

Request Validation:
OtpController receives an HTTP POST request.
Calls OtpValidator.validateOtpRequest to validate the request.
OTP Generation:
OtpService.generateOtp generates a random OTP.
Saves OTP details to the OTP_MANAGEMENT table.
Sends the OTP to the recipient using NotificationUtil.
Database Interaction:
OtpRepository handles database operations to save or retrieve OTP records.
Response:
A success message is sent back to the client.
Sample Request
Endpoint: POST /api/otp/generate

Request Body:

{
    "userId": "12345",
    "requestType": "login",
    "channel": "EMAIL",
    "recipient": "user@example.com"
}
Response:

OTP successfully generated and sent via EMAIL.
Testing the Flow
Start the application.
Test using tools like Postman.
Verify:
OTP generation.
Email/SMS mock output.
Database entry in the OTP_MANAGEMENT table.
Let me know if you need further customization!





You said:
How can i send the otp generated by my code via notification utility class based on sms and email service 
ChatGPT said:

ChatGPT
To send the OTP generated by your code via SMS or Email services using a NotificationUtility class, you can follow this structured approach. I'll provide a step-by-step implementation to integrate SMS and Email notification services.

1. NotificationUtility Class

The NotificationUtility will act as a centralized utility to send notifications. It will use:

An Email Service for sending OTP via email.
An SMS Service for sending OTP via SMS.
Implement NotificationUtility
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class NotificationUtility {

    @Autowired
    private EmailService emailService;

    @Autowired
    private SmsService smsService;

    // Unified method to send OTP based on channel
    public void sendOtp(String channel, String recipient, String otpCode) {
        if ("EMAIL".equalsIgnoreCase(channel)) {
            emailService.sendEmail(recipient, otpCode);
        } else if ("SMS".equalsIgnoreCase(channel)) {
            smsService.sendSms(recipient, otpCode);
        } else {
            throw new IllegalArgumentException("Invalid channel. Supported channels are EMAIL or SMS.");
        }
    }
}
2. Email Service

A dedicated service to send OTP via email. You can use libraries like JavaMailSender to send emails.

Implement EmailService
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Service;

import javax.mail.MessagingException;
import javax.mail.internet.MimeMessage;

@Service
public class EmailService {

    @Autowired
    private JavaMailSender mailSender;

    public void sendEmail(String recipient, String otpCode) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);

            helper.setTo(recipient);
            helper.setSubject("Your OTP Code");
            helper.setText("Dear User,\n\nYour OTP Code is: " + otpCode + "\n\nThis code is valid for 5 minutes.\n\nThank you!");

            mailSender.send(message);
            System.out.println("Email sent successfully to " + recipient);

        } catch (MessagingException e) {
            throw new RuntimeException("Failed to send OTP via email.", e);
        }
    }
}
3. SMS Service

A dedicated service to send OTP via SMS. You can use third-party providers like Twilio, Nexmo, or any SMS gateway API.

Mock SmsService (Replace this with actual SMS API implementation)
import org.springframework.stereotype.Service;

@Service
public class SmsService {

    public void sendSms(String phoneNumber, String otpCode) {
        // Replace this with actual SMS API integration
        System.out.println("Sending OTP [" + otpCode + "] to phone number: " + phoneNumber);

        // Example integration with SMS gateway:
        // smsGatewayApi.send(phoneNumber, "Your OTP code is: " + otpCode);
    }
}
4. Integrate NotificationUtility in OTP Service

Now, call the NotificationUtility class from your OtpService to send OTP.

Update OtpService
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.Random;

@Service
public class OtpService {

    private static final int OTP_EXPIRY_MINUTES = 5;

    @Autowired
    private NotificationUtility notificationUtility;

    @Autowired
    private OtpRepository otpRepository;

    public String generateOtp(String userId, String requestType, String channel, String recipient) {
        // Step 1: Generate a random OTP
        String otpCode = generateRandomOtp();

        // Step 2: Calculate expiry time
        Date expiryTime = new Date(System.currentTimeMillis() + OTP_EXPIRY_MINUTES * 60 * 1000);

        // Step 3: Save OTP details into database
        saveOtpToDatabase(userId, requestType, otpCode, expiryTime, channel);

        // Step 4: Send OTP via NotificationUtility
        notificationUtility.sendOtp(channel, recipient, otpCode);

        return "OTP successfully generated and sent via " + channel;
    }

    private String generateRandomOtp() {
        return String.format("%06d", new Random().nextInt(999999));
    }

    private void saveOtpToDatabase(String userId, String requestType, String otpCode, Date expiryTime, String channel) {
        OtpEntity otpEntity = new OtpEntity();
        otpEntity.setUserId(userId);
        otpEntity.setRequestType(requestType);
        otpEntity.setOtpCode(otpCode);
        otpEntity.setExpiryTime(expiryTime);
        otpEntity.setIsVerified(false);
        otpEntity.setChannel(channel);
        otpEntity.setCreatedDate(new Date());

        otpRepository.save(otpEntity);
    }
}
5. Update OTP Controller

Ensure your controller calls the OTP generation logic correctly.

Update OtpController
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/otp")
public class OtpController {

    @Autowired
    private OtpValidator otpValidator;

    @Autowired
    private OtpService otpService;

    @PostMapping("/generate")
    public ResponseEntity<String> generateOtp(@RequestBody OtpRequest request) {
        // Validate the request
        otpValidator.validateOtpRequest(request.getRequestType(), request.getChannel());

        // Generate and send OTP
        String responseMessage = otpService.generateOtp(
            request.getUserId(),
            request.getRequestType(),
            request.getChannel(),
            request.getRecipient()
        );

        return ResponseEntity.ok(responseMessage);
    }
}
6. Application Properties for Email

For email functionality, configure your SMTP server in application.properties.

spring.mail.host=smtp.example.com
spring.mail.port=587
spring.mail.username=your_email@example.com
spring.mail.password=your_password
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
Replace these values with your SMTP server details.

7. Test the API

Endpoint: POST /api/otp/generate
Request Body:
{
    "userId": "12345",
    "requestType": "login",
    "channel": "EMAIL",
    "recipient": "user@example.com"
}
For SMS, set "channel": "SMS" and "recipient": "9876543210".
Response:
"OTP successfully generated and sent via EMAIL."
Verify:
For EMAIL: Check the console or inbox for the OTP message.
For SMS: Check the console (mock implementation prints the OTP to the console).
Summary

NotificationUtility: Delegates the OTP notification to either the Email or SMS service.
EmailService: Sends OTP via email using JavaMailSender.
SmsService: Sends OTP via SMS (mock implementation; replace with actual API).
OtpService: Generates OTP, saves it to the database, and calls NotificationUtility.
Controller: Validates input and triggers the OTP generation.
This approach separates concerns and allows easy testing and extension for different notification channels. Let me know if you need further assistance!





You said:
Instead of below if condition we want to implement below regrex class logic to validate userid  

if (otpGenerationRequest.getUserId().contains(String.valueOf('@'))) { EmailDTO emailDTO = new EmailDTO(); isOtpSend = emailService.sendEmail(emailDTO);

    } else {
        try {
            SmsDTO smsDTO = new SmsDTO();
            isOtpSend = smsService.sendSMS(smsDTO);

        }
        catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
import java.util.regex.Pattern;

public class RegexValidation {

// Regex for validating phone number
private static final String PHONE_REGEX = "^\\+?[0-9]{10,15}$";

// Regex for validating email
private static final String EMAIL_REGEX = "^[A-Za-z0-9+_.-]+@(.+)$";

// Validate phone number
public static boolean validatePhone(String phone) {
    return phone != null && Pattern.matches(PHONE_REGEX, phone);
}

// Validate email
public static boolean validateEmail(String email) {
    return email != null && Pattern.matches(EMAIL_REGEX, email);
}

// Validate user ID (alphanumeric check)
public static boolean validateUserId(String userId) {
    return userId != null && userId.matches("^[a-zA-Z0-9]+$");
}
}
ChatGPT said:

ChatGPT
To implement the RegexValidation class logic for validating the user ID in place of the if-else condition, follow these steps:

1. Integrate RegexValidation in Your Service

Replace the if condition logic with calls to the RegexValidation class to determine whether the user ID is an email or a phone number.

Updated OTP Service Logic:
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class OtpService {

    @Autowired
    private EmailService emailService;

    @Autowired
    private SmsService smsService;

    public boolean sendOtpBasedOnUserId(OtpGenerationRequest otpGenerationRequest) {
        boolean isOtpSend = false;

        String userId = otpGenerationRequest.getUserId();

        if (RegexValidation.validateEmail(userId)) {
            // Send Email
            EmailDTO emailDTO = new EmailDTO();
            emailDTO.setRecipient(userId); // Set recipient email
            emailDTO.setOtpCode("123456"); // Replace with actual OTP
            isOtpSend = emailService.sendEmail(emailDTO);
        } else if (RegexValidation.validatePhone(userId)) {
            // Send SMS
            SmsDTO smsDTO = new SmsDTO();
            smsDTO.setPhoneNumber(userId); // Set recipient phone number
            smsDTO.setOtpCode("123456");   // Replace with actual OTP
            try {
                isOtpSend = smsService.sendSMS(smsDTO);
            } catch (Exception e) {
                throw new RuntimeException("Failed to send OTP via SMS", e);
            }
        } else {
            throw new IllegalArgumentException("Invalid User ID. It should be a valid email or phone number.");
        }

        return isOtpSend;
    }
}
2. Update the Controller to Use the New Logic

Call the sendOtpBasedOnUserId method from the controller.

Updated Controller:
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/otp")
public class OtpController {

    @Autowired
    private OtpService otpService;

    @PostMapping("/generate")
    public ResponseEntity<String> generateOtp(@RequestBody OtpGenerationRequest request) {
        // Attempt to send OTP
        boolean isOtpSend = otpService.sendOtpBasedOnUserId(request);

        if (isOtpSend) {
            return ResponseEntity.ok("OTP successfully sent.");
        } else {
            return ResponseEntity.status(500).body("Failed to send OTP.");
        }
    }
}
3. OtpGenerationRequest DTO

Ensure that the request DTO captures the userId.

public class OtpGenerationRequest {
    private String userId;

    // Getters and setters
    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }
}
4. EmailDTO and SmsDTO

Ensure these DTOs are properly defined to carry data for email and SMS notifications.

EmailDTO:
public class EmailDTO {
    private String recipient;
    private String otpCode;

    // Getters and setters
    public String getRecipient() {
        return recipient;
    }

    public void setRecipient(String recipient) {
        this.recipient = recipient;
    }

    public String getOtpCode() {
        return otpCode;
    }

    public void setOtpCode(String otpCode) {
        this.otpCode = otpCode;
    }
}
SmsDTO:
public class SmsDTO {
    private String phoneNumber;
    private String otpCode;

    // Getters and setters
    public String getPhoneNumber() {
        return phoneNumber;
    }

    public void setPhoneNumber(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    public String getOtpCode() {
        return otpCode;
    }

    public void setOtpCode(String otpCode) {
        this.otpCode = otpCode;
    }
}

package com.epay.merchant.service;

import com.epay.merchant.entity.NotificationEntity;
import com.epay.merchant.entity.Otp;
import com.epay.merchant.exception.MerchantException;
import com.epay.merchant.model.request.OtpGenerationRequest;
import com.epay.merchant.model.response.OtpGenerationResponse;
import com.epay.merchant.model.response.ResponseDto;
import com.epay.merchant.repository.NotificationRepository;
import com.epay.merchant.repository.OtpRepository;
import com.epay.merchant.util.AppConstants;
import com.epay.merchant.util.ErrorConstants;
import com.epay.merchant.util.RegexValidation;
import com.epay.merchant.validator.OtpGenerationValidator;
import com.sbi.epay.notification.model.EmailDTO;
import com.sbi.epay.notification.model.SmsDTO;
import com.sbi.epay.notification.service.EmailService;
import com.sbi.epay.notification.service.SmsService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.text.MessageFormat;
import java.util.List;
import java.util.Random;
import java.util.UUID;



@Service
@RequiredArgsConstructor
public class OtpService {

    private final OtpRepository otpRepository;
    private final EmailService emailService;
    private final SmsService smsService;
    private final NotificationRepository notificationRepository;
    private boolean isOtpSend = false;
    private final OtpGenerationValidator otpGenerationValidator;
    private final RegexValidation regexValidation;


    public ResponseDto<OtpGenerationResponse> generateOtp(OtpGenerationRequest otpGenerationRequest) {
        otpGenerationValidator.validateOtpRequest(otpGenerationRequest);


        String requestOtp = String.valueOf(new Random().nextInt(900000) + 100000); // Generate 6-digit OTP
        Long currentTimeMillis = System.currentTimeMillis();
        Long expiryTimeMillis = currentTimeMillis + (5 * 60 * 1000);
        Otp otp = new Otp();
        otp.setRequestType(otpGenerationRequest.getRequestType());
        otp.setUserId(otpGenerationRequest.getUserId());
        otp.setRequestID(UUID.randomUUID());
        otp.setOtpCode(requestOtp);
        otp.setExpiryTime(expiryTimeMillis);
        otp.setIsVerified(false);
        otp.setCreateDate(currentTimeMillis);
        otp.setUpdateDate(currentTimeMillis);

        // Save to DB
        otp = otpRepository.save(otp);

        //TODO to call producer and KAFKA Implementation
        //call utility


        String userId = otpGenerationRequest.getUserId();
        if (RegexValidation.validateEmail(userId)) {
            // Send Email
            //  if (otpGenerationRequest.getUserId().contains(String.valueOf('@'))) {
            EmailDTO emailDTO = new EmailDTO();
            emailDTO.setRecipient(userId);
            emailDTO.setBody(requestOtp);
            isOtpSend = emailService.sendEmail(emailDTO);

        } else if (RegexValidation.validatePhone(userId)) {
            try {
                SmsDTO smsDTO = new SmsDTO();
                smsDTO.setMobileNumber(userId);
                smsDTO.setMessage(requestOtp);
                isOtpSend = smsService.sendSMS(smsDTO);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }

            if (isOtpSend) {
                NotificationEntity notificationEntity = new NotificationEntity();
                notificationEntity.setENTITY_NAME(otpGenerationRequest.getRequestType());
                notificationEntity.setTYPE("OTP");
                notificationEntity.setMODE_TYPE(otpGenerationRequest.getUserId().contains(String.valueOf('@')) ? "EMAIL" : "SMS");
                notificationEntity.setVALUE("");
                notificationEntity.setSTATUS("Completed");
                notificationEntity.setCREATED_AT(currentTimeMillis);
                notificationEntity.setCREATED_BY(otpGenerationRequest.getUserId());

                notificationEntity = notificationRepository.save(notificationEntity);

                OtpGenerationResponse otpGenerationResponse = new OtpGenerationResponse(otp.getUserId(), "OTP successfully generated for user: " + otp.getUserId());

                return ResponseDto.<OtpGenerationResponse>builder()
                        .status(AppConstants.RESPONSE_SUCCESS)
                        .data(List.of(otpGenerationResponse))
                        .count(1L)
                        .total(1L)
                        .build();
            }
            OtpGenerationResponse errorResponse = new OtpGenerationResponse(null, "failed to send otp");

            return ResponseDto.<OtpGenerationResponse>builder()
                    .status(AppConstants.RESPONSE_FAILURE)
                    .data(List.of(errorResponse))
                    .count(1L)
                    .total(1L)
                    .build();

        }

        throw new MerchantException(ErrorConstants.INVALID_ERROR_CODE, MessageFormat.format(ErrorConstants.INVALID_ERROR_MESSAGE, "User Id"));
    }
}

    package com.epay.merchant.validator;
import com.epay.merchant.dto.OtpRequest;
import com.epay.merchant.exception.MerchantException;
import com.epay.merchant.model.request.OtpGenerationRequest;
import com.epay.merchant.util.ErrorConstants;
import com.epay.merchant.util.enums.RequestType;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.text.MessageFormat;
import java.util.logging.ErrorManager;


@Component
@RequiredArgsConstructor
public class OtpGenerationValidator extends BaseValidator {
   // private static final LoggerUtility logger = LoggerFactoryUtility.getLogger(OtpRequest.class);
//   /* private final OtpRequest otpRequest;*/


    public void validateOtpRequest(OtpGenerationRequest otpGenerationRequest) {
        //logger.debug("Customer validation start for {}", otpRequest);
        validateMandatoryFields(otpGenerationRequest);
       // logger.debug("Customer mandatory validation completed for {}", otpRequest);
        validateEnumValue(otpGenerationRequest.getRequestType());
       // logger.debug("Customer field validation completed for {}", otpRequest);
    }

    private void validateMandatoryFields(OtpGenerationRequest otpGenerationRequest) {
        checkMandatoryField(otpGenerationRequest.getRequestType(), "Request type");
        checkMandatoryField(otpGenerationRequest.getUserId(), "Userid");
        throwIfErrors();
    }

    private void validateEnumValue(String value) {
        if (!(value.equals(RequestType.LOGIN.getValue()) || value.equals(RequestType.CHANGE_PASSWORD.getValue()) || value.equals(RequestType.RESET_PASSWORD.getValue()))) {
            throw new MerchantException(ErrorConstants.INVALID_ERROR_CODE,MessageFormat.format(ErrorConstants.INVALID_ERROR_MESSAGE, "Request Type"));
        }
    }


}

