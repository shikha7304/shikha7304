public final class SmsService {
    LoggerUtility logger = LoggerFactoryUtility.getLogger(SmsService.class);
    private final SmsClient smsClient;

    /**
     * This method will be used for validating smsDTO object and send sms
     *
     * @param smsDTO passing as parameter for send sms
     * @return boolean true or false
     * @throws NotificationException if any exception occur
     */
    public boolean sendSMS(SmsDTO smsDTO) throws JsonProcessingException, NotificationException {
        logger.info("ClassName - SmsService,MethodName -sendSMS, Method-Start");
        SmsValidator.validateSMS(smsDTO);
        logger.info("ClassName - SmsService,MethodName -sendSMS, Method-end");
        return smsClient.sendSMS(smsDTO);
    }

}


public class SmsClient extends ApiClient {

    private final ObjectMapper objectMapper;
    Logger logger = LoggerFactory.getLogger(this.getClass().getName());

    /**
     * This constructor used for baseUrl and objectMapper
     *
     * @param baseUrl for sms
     */
    public SmsClient(@Value("${sms.gateway.url}") String baseUrl) {
        super(baseUrl);
        this.objectMapper = new ObjectMapper();
    }


    /**
     * This method will be used for sending sms
     *
     * @param smsDTO for sending sms
     * @return boolean true or false
     * @throws NotificationException   if any exception occurs
     * @throws JsonProcessingException if any exception occurs
     */
    public boolean sendSMS(SmsDTO smsDTO) throws NotificationException, JsonProcessingException {
        logger.info("ClassName - SmsClient,MethodName - sendSMS,Method-start");

        String smsRequest = objectMapper.writeValueAsString(smsDTO);
        HttpEntity<String> requestEntity = prepareHttpEntity(smsRequest, prepareHttpHeaders());
        ResponseEntity<String> response = getRestTemplate().exchange(getBaseUrl(), HttpMethod.POST, requestEntity, String.class);
        if (response.getStatusCode().is2xxSuccessful()) {
            return true;
        } else {
            throw new NotificationException(NotificationConstant.FAILURE_CODE, MessageFormat.format(NotificationConstant.FAILURE_MSG, "SMS"));
        }

    }
}


public class SmsValidator {

    static LoggerUtility logger = LoggerFactoryUtility.getLogger(SmsValidator.class);
    /**
     * This method will be used for validating the sms
     *
     * @param smsRequest for validate sms
     * @throws NotificationException if any exception occurs
     */
    public static void validateSMS(SmsDTO smsRequest) throws NotificationException {
        logger.info("ClassName - SMSValidator,MethodName - validateSMS, method-start");
        List<ErrorDto> errorList = new ArrayList<>();

        if (StringUtils.isEmpty(smsRequest.getMessage())) {
            errorList.add(new ErrorDto(NotificationConstant.MANDATORY_ERROR_CODE, MessageFormat.format(NotificationConstant.MANDATORY_ERROR_MESSAGE, "SMS Message")));
        }

        if (StringUtils.isEmpty(smsRequest.getMobileNumber())) {
            errorList.add(new ErrorDto(NotificationConstant.MANDATORY_ERROR_CODE, MessageFormat.format(NotificationConstant.MANDATORY_ERROR_MESSAGE, "Mobile Number")));
        }

        /**
         *  this method of Collection.isEmpty check if
         *  the error list have some error
         *  then it throw exception
         **/
        if (!CollectionUtils.isEmpty(errorList)) {
            logger.error("Error -> ",errorList);
            throw new NotificationException(errorList);
        }
        logger.info("ClassName - SMSValidator,MethodName - validateSMS, method-end");
    }
}
