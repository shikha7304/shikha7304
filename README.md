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

        
      OTP should expire after 5 minutes. 

        
      Validation should confirm the user-provided OTP matches the stored OTP. 

        
      Users can resend the OTP if needed.

        
      All requests and responses must be logged for traceability. 
