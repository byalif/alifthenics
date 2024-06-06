# Project Name: Training program

## Introduction

alifthenics.com is a fitness platform for clients to sign up for paid programs as well as one-to-one training. This tool integrates with various services like Kafka and the Stripe API to create fast transaction receipts emailed to clients after a purchase or inquiry. The website is designed using a robust microservices architecture, ensuring scalability and efficient management of different service components.

## Project Architecture

### Client Side:

- React.js is used in the frontend part of this project.

### API Gateway:

- Spring Cloud Gateway is used to act as the single-entry point for all client requests to the backend services. It routes requests to the appropriate microservices and provides common functionalities like authentication and logging.

### Microservices:

- The application is split into microservices that can be set up on their own, making it simpler to manage and allowing for easier updates and scalability.
- Services include Stripe integration, Mailer service, Kafka consumer, Kafka producer, Question inquiries, Quiz service, API Gateway, Authentication...

### Database:

- MySQL is used for each microservice, with plans to migrate to MongoDB for some services.

### Cache Mechanism:

- Redis cache is used as an in-memory cache solution for storing frequently accessed data such as user products.

### Logging tools:

- Log4j2 is used for powerful logging capabilities, integrated with Zipkin for real-time log analysis.

### Messaging system:

- Kafka is used for real-time data updating and asynchronous communications.

### Security:

- Spring security on the API gateway is implemented for secure access control instead of putting that on every microservice.

### Deployment and operations:

- Docker is used for containerization, Kubernetes for orchestration, and Jenkins for CI/CD.
- Github is used for versioning.

## Project Flow

- Detailed steps include User Interaction and Authentication, Inquiry Collection and Processing, Payment processing, and program generation and Delivery.
- All interactions are logged and can be seen on Zipkin server.
- Kafka producers and consumers are used for asyncronous communication between services.

## Data Layer / Schemas:

- Major Databases include securityDB, questionsDB, quizDB, transactionsDB, and emailDB.

## Some important snippets of code

- This code below defines a custom authentication filter for a Spring Cloud Gateway, which is a common component in a microservice architecture. The filter checks for the presence and validity of a JWT token in HTTP requests, extracts user information from the token, and verifies the user's role against the requested URI. Here's a summary and the importance of each part in a microservice architecture:

```java

 @Override
    public GatewayFilter apply(Config config) {
        return ((exchange, chain) -> {
           	ServerHttpRequest request = null;
            if (validator.isSecured.test(exchange.getRequest())) {
                //header contains token or not
                if (!exchange.getRequest().getHeaders().containsKey(HttpHeaders.AUTHORIZATION)) {
                    throw new JwtException("MISSING_TOKEN");
                }

                String authHeader = exchange.getRequest().getHeaders().get(HttpHeaders.AUTHORIZATION).get(0);

                if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                	throw new JwtException("INVALID_TOKEN");
                }

                authHeader = authHeader.substring(7);

        		try {
        			jwtUtil.validateToken(authHeader);
        		} catch (JwtException e) {
        			throw new JwtException("INVALID_TOKEN");
        		}

        		String[] extracted = jwtUtil.extractUsername(authHeader).split(" ");
        		String username = extracted[0];
        		String role = "ROLE_"+ extracted[1];

        		String reqUri = exchange.getRequest().getURI().getPath();

        		boolean authorized= myFilter.hasRequiredRole(role, reqUri);
        		if(!authorized) throw new JwtException("UNAUTHORIZED");

                request = exchange.getRequest().mutate().header("username",username).build();



            }
            return chain.filter(exchange.mutate().request(request).build());
        });
    }
```

- This code defines a method createQuiz in a service that communicates with another microservice to create a new quiz.
  Microservices can be scaled independently based on load and resource requirements.
  This method allows the quiz-service-svc to handle quiz creation, while other services can handle different responsibilities.

```java
@Override
	public Quiz createQuiz(List<Question> questions, String name) {
		// Set up HTTP headers
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);

        // Set up the request body
        HttpEntity<List<Question>> requestEntity = new HttpEntity<>(questions, headers);

        // Send the POST request
        ResponseEntity<Quiz> responseEntity = restTemplate.exchange(
            "http://quiz-service-svc/quiz/create/newQuiz/{name}",
            HttpMethod.POST,
            requestEntity,
            Quiz.class,
            name
        );

        // Return the response body
        return responseEntity.getBody();
	}

```

- This code defines a method newClientSubmission that handles the submission of answers to a fitness-related questionnaire. It sends this information to a Kafka topic for further processing. By sending the questionnaire data to Kafka, the system can handle the processing of client submissions asynchronously. This decouples the client submission from the processing logic, making the system more responsive and efficient.

```java
@Override
	public ResponseEntity<ResponseDTO> newClientSubmission(QuestionDTO questions) {
		try {
            // Publish a message to Kafka topic containing necessary information
			KafkaDTO kafkaDTO = new KafkaDTO();
			kafkaDTO.setQuestionDTO(questions);
            kafkaProducer.sendMessage(kafkaDTO);
            return ResponseEntity.status(HttpStatus.OK).body(new ResponseDTO("Email request sent to Kafka topic: "+ questions.getEmail()));
        } catch (Exception e) {
        	// catch this later
            log.error("Failed to send email request to Kafka topic", e);
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(new ResponseDTO("Failed to send email request to Kafka topic"));
        }
	}

```

- The provided code defines two Kafka consumers for my fitness app, which listen to different topics and process incoming messages. Both listeners handle tasks asynchronously, ensuring that email sending and database operations do not block the main application thread.
  This improves the responsiveness and performance of the application.

```java
	@KafkaListener(topics = {"transaction_requests"}, groupId = "email")
	public void transactionListener(KafkaDTO kafkaDTO) {
		log.info(String.format("New Transaction from: ", kafkaDTO.getEmailDto().getEmail()));

        // Send email asynchronously
        emailService.sendEmailAsync(kafkaDTO.getEmailDto());

        emailService.sendAddtional(kafkaDTO.getEmailDto().getEmail());

        //Save the transaction to DB

        Receipt receipt = emailService.saveInvoiceToDB(kafkaDTO.getEmailDto());

		log.info(String.format("Transaction id: %d", receipt.getId()));
    }

	@KafkaListener(topics = {"inquiry_requests"}, groupId = "email")
	public void inquiryListener(KafkaDTO kafkaDTO) {
		log.info(String.format("New client Inquiry: ", kafkaDTO.getQuestionDTO().getEmail()));

        // Send email asynchronously
        emailService.sendEmailAsync(kafkaDTO.getQuestionDTO());


        Inquiry inquiry = emailService.saveInquiryToDB(kafkaDTO.getQuestionDTO());

        log.info(String.format("Inquiry id: ", inquiry.getId()));
        }
```
