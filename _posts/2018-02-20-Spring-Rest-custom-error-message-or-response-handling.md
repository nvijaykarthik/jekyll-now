
#### Spring Rest Custom Error Code(HTTP)/Message Response

## Problem Statement
In ReST Webservice World. Responding with the Http Error codes for the exception thrown in java is not easier , and that too with the customer error messages for that particular exception also needs lots of configuration.
Spring provides certain features that can be leverage to respond the exceptions thrown in java with the custom error message and HTTP error codes.
let us see how it can be implemented.
	
## Solution
Solution to the above statement is simple , the error code and the messages are configured in the property file and for particular exception we can configure the application to throw the particular error message and HTTP status code.

Before going into solution we assume that the reader posses  java, spring boot,Maven or Gradle and ReST WS knowledge. 

**Setting up the project** 
	- Create a spring boot project using the below link.
	[Spring initializer](https://start.spring.io/)
	- Import the project into STS or any developer tool that supports the spring boot.
	once your project is imported , just run and check whether the dependencies are downloaded and project is running. Please use the following link to set up basic spring boot web project 
	[Building a RESTful Web Service](https://spring.io/guides/gs/rest-service/)
Let us look into to the solution which is in our scope ie., error / exception handling.

Create below packages that holds the objects 

> Package to hold the custom exceptions related to application
> Package to expose the Web Resource ie., ReST controller
> Package to handle the Exception 
> Package to hold the domain object ie., in this case Error response

### Exception Mapping 
In the package that handled the exception create a class that holds the Error code and Error message mappings from properties
Code below

```
@Component
@ConfigurationProperties(prefix="vijay")
public class ExceptionMapping {

	private Map<Class<? extends Throwable>, String> errorMessageMapping=new HashMap<>();
	private Map<Class<? extends Throwable>, String> errorCodeMapping=new HashMap<>();

	public Map<Class<? extends Throwable>, String> getErrorCodeMapping() {
		return errorCodeMapping;
	}

	public void setErrorCodeMapping(Map<Class<? extends Throwable>, String> errorCodeMapping) {
		this.errorCodeMapping = errorCodeMapping;
	}

	public Map<Class<? extends Throwable>, String> getErrorMessageMapping() {
		return errorMessageMapping;
	}

	public void setErrorMessageMapping(Map<Class<? extends Throwable>, String> errorMessageMapping) {
		this.errorMessageMapping = errorMessageMapping;
	}


}
```
class ExceptionMapping has two Map variable that holds the exception class , message and code .

This class it annotated with @ConfigurationProperties where this annotation exposes the variable that has getter and setter methods to the application properties , where you can configure the exception and code/message.
We prefix with vijay to group these props under vijay.

### Exception Control Advice
In the package that handled the exception create a class that holds the handles the exception and responds with the http error code and error messages 
Code below
```
@ControllerAdvice
public class ExceptionControllerAdvice {

	private static final Logger log = LoggerFactory.getLogger(ExceptionControllerAdvice.class.getName());
	private HttpStatus status = HttpStatus.SERVICE_UNAVAILABLE;

	@Autowired
	ExceptionMapping mapping;

	@ExceptionHandler(Throwable.class)
	public ResponseEntity<ExceptionMessage> exceptionHandler(Throwable ex) {

		Class<? extends Throwable> clazz = ex.getClass();

		Integer customErrorCode = null != mapping.getErrorCodeMapping().get(clazz)
				? Integer.parseInt(mapping.getErrorCodeMapping().get(clazz))
				: 503;
		String customMessage = mapping.getErrorMessageMapping().get(clazz);

		ExceptionMessage exceptionMessage = new ExceptionMessage();

		exceptionMessage.setErrorCode(customErrorCode);
		status = HttpStatus.valueOf(customErrorCode);

		if (null != customMessage) {
			exceptionMessage.setErrorMessage(customMessage);

		} else {
			String msg = ex.getLocalizedMessage();
			exceptionMessage.setErrorMessage(msg);
		}

		return new ResponseEntity<ExceptionMessage>(exceptionMessage, status);
	}
}
```
The class ExceptionControllerAdvice is annotated with @ControllerAdvice which means that when ever there is a error or exception please execute this class

you can create many methods within this class and handle the exceptions specifically by annotating the method with @ExceptionHandler
Since our solution is generic we have one method that handles all the exception so we annotated with @ExceptionHandler(Throwable.class)
why Throwable? because all the java error/exception comes under this so you can handle all the exceptions.

The implementation logic is simple.

1. the method holds the Throwable as param  the exception instance is assign to this param
2. Get the class for that error or exception
3. Get the exception handler instance and get the configured code or message from the Map.
4. Now check whether the code or message is configured to that particular exception if not throw Service unavailable error code and Exception message from exception. 

### Rest Controller 
In the package that exposes the web service create a class that exposes a simple ReST controller that throw the custom or lang exceptions
```
@RestController
@RequestMapping("/api")
public class ReSTController {

	@RequestMapping("/throwCustomException")
	public void throwCustomeExeption() throws CustomException {
		throw new CustomException("Throwing CUstom Exception");
	}
	
	@RequestMapping("/throwBadReqException")
	public void throwBadReqExeption() throws IllegalArgumentException {
		throw new IllegalArgumentException("Throwing CUstom Exception");
	}
}
```

In this Controller both methods throw particular exception.

### Error code and Error Message Mapping
In application.yml or properties ( we use YML for easy understanding and grouping the properties)
Configure the yml file to throw particular error or message.
```
vijay:
  error-code-mapping:
    java.lang.IllegalArgumentException: 401
    io.vijaykarthik.exception.CustomException: 402
  error-message-mapping:
    java.lang.IllegalArgumentException: Invalid Input
    io.vijaykarthik.exception.CustomException: Custom error message
```

Please try this and let us know.

[Get the working example ](https://github.com/nvijaykarthik/Spring-Rest-Custom-Error-Message-Response)
