public class NotificationException extends RuntimeException {

    private String errorCode;
    private String errorMessage;
    private List<ErrorDto> errorMessages;

    /**
     * This is parametrised constructor for taking errorCode and  errorMessage
     *
     * @param errorCode
     * @param errorMessage
     */
    public NotificationException(String errorCode, String errorMessage) {
        super(errorMessage);
        this.errorCode = errorCode;
        this.errorMessage = errorMessage;
    }

    /**
     * This constructor used for list of errorMessages
     *
     * @param errorMessages
     */
    public NotificationException(List<ErrorDto> errorMessages) {
        this.errorMessages = errorMessages;
    }
}
