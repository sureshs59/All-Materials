
Explain REST and the difference between REST and SOAP.
===================================================

What is REST?

REST means:

Representational State Transfer

It is an architectural style for building web services.

REST typically uses:

HTTP protocol
URI-based resources
JSON (most common), sometimes XML

A resource could be:

/customers
/orders
/payments
/products

Example:

GET /api/customers/101

Response:

{
  "id": 101,
  "name": "John"
}
REST Principles
1. Client-Server

Frontend and backend are separated.

Example:

Angular frontend ↔ Spring Boot backend
2. Stateless

Every request contains all required information.

Server does not remember previous request state.

Example:

Authorization: Bearer token

Each request carries authentication.

3. Resource-Based

Everything is treated as a resource.

Examples:

/users
/accounts
/orders
4. Uses HTTP Methods
Method	Purpose
GET	Read
POST	Create
PUT	Update
PATCH	Partial update
DELETE	Remove
Example REST API in Spring Boot
@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    @GetMapping("/{id}")
    public Customer getCustomer(
            @PathVariable Long id
    ) {
        return service.findById(id);
    }

    @PostMapping
    public Customer createCustomer(
            @RequestBody Customer customer
    ) {
        return service.save(customer);
    }
}


What is SOAP?

SOAP means:

Simple Object Access Protocol

SOAP is a protocol for exchanging structured messages.

SOAP usually uses:

XML only
WSDL contract
strict standards

SOAP message example:

<soap:Envelope>
   <soap:Body>
      <getCustomer>
         <id>101</id>
      </getCustomer>
   </soap:Body>
</soap:Envelope>


REST										SOAP
Architectural style						Protocol
Lightweight								Heavy
Mostly JSON								XML only
Faster										Slower
Easy to consume						More complex
Flexible										Strict contract
Uses HTTP mainly						HTTP, SMTP, TCP
No built-in security						WS-Security support
Best for microservices				Best for enterprise legacy systems


When Would You Use REST?

Use REST for:

web applications
mobile apps
microservices
cloud APIs

Example:

Angular + Spring Boot
When Would You Use SOAP?

Use SOAP for:

banking systems
healthcare
government integrations
legacy enterprise systems

Because:

WS-Security
formal contracts
guaranteed messaging
Senior Interview Answer

REST is an architectural style used for building lightweight, stateless web services where resources are accessed using HTTP methods such as GET, POST, PUT, and DELETE. It commonly uses JSON and is widely used in microservices and modern web applications. SOAP, on the other hand, is a protocol that uses XML messages and a WSDL contract. SOAP provides built-in standards for security, transactions, and reliable messaging, which makes it common in banking, healthcare, and legacy enterprise integrations. In my recent projects, I mainly used REST with Angular and Spring Boot for scalable microservices, while SOAP was used when integrating with legacy external enterprise systems.



==================================================================================



Since you were working on improving response times and thread efficiency in that migration, how do you approach debugging a 
performance issue in a Java application, and what tools or methodologies do you typically use?

When I debug a Java performance issue, I first avoid guessing and start with data. I look at where the slowdown is happening: application code, database, external API, thread pool, memory, or infrastructure.

My first step is to check monitoring dashboards such as CloudWatch, AppDynamics, Splunk, or Datadog. I look at response time, throughput, error rate, CPU, memory, GC activity, thread count, and database latency.

Then I check logs and traces for slow endpoints. If the issue is API latency, I identify whether the time is spent inside business logic, database queries, or downstream service calls. For distributed systems, I use correlation IDs or tracing to follow the request across services.

If I suspect thread issues, I take thread dumps using tools like jstack, VisualVM, or JDK Mission Control. I look for blocked threads, deadlocks, thread starvation, or too many threads waiting on external calls.

If I suspect memory or GC issues, I review GC logs and heap usage. I use tools like VisualVM, Eclipse MAT, JConsole, or JDK Mission Control to analyze heap dumps and identify memory leaks, large object retention, or frequent full GCs.

For database-related performance problems, I review slow queries, execution plans, missing indexes, N+1 query problems, connection pool saturation, and transaction boundaries.

For code-level issues, I use profilers like YourKit, VisualVM, or JFR to identify hot methods, expensive loops, unnecessary object creation, or blocking calls.

Once I identify the bottleneck, I apply targeted fixes such as query optimization, caching, async processing, WebClient parallel calls, connection pool tuning, thread pool tuning, or Resilience4j timeout/circuit breaker settings. After that, I validate improvements using load testing and compare before/after metrics like P95 latency and throughput.

Short version:

I approach Java performance debugging by first identifying the bottleneck using metrics, logs, traces, thread dumps, heap dumps, GC logs, and database execution plans. I use tools like AppDynamics, CloudWatch, Splunk, VisualVM, JFR, JConsole, Eclipse MAT, and jstack. Then I apply targeted fixes and validate with load testing and P95/P99 latency metrics.

Interview-ready closing line:

My main methodology is measure first, isolate the bottleneck, apply a focused fix, and validate with before-and-after metrics.

=======================================================================================

You mentioned converting legacy systems to modern reactive applications earlier. Can you describe that application in more detail and explain the key challenges you faced during its development?

In one of my projects, we modernized a legacy application that had a traditional synchronous request-response architecture. The older system used blocking service calls and had performance issues when multiple downstream APIs were called sequentially. Some dashboard and transaction screens were taking more time to load because each API call waited for the previous one to complete.

We redesigned the application using a modern Angular frontend and Spring Boot backend. On the frontend, we used Angular with RxJS to handle API calls more efficiently. For independent calls, we used forkJoin to execute multiple requests in parallel. For dependent calls, we used switchMap or concatMap, depending on whether we needed cancellation or ordered execution.

On the backend, we moved toward Spring Boot microservices and used WebClient for non-blocking service-to-service communication. We also added Resilience4j for retry, timeout, circuit breaker, and fallback handling so one failing downstream service would not impact the entire application.

One key challenge was converting sequential blocking flows into reactive flows without breaking business logic. We had to carefully identify which calls were independent and which calls were dependent. Another challenge was error handling. In a parallel flow, if one API failed, the whole response could fail, so we used fallback handling and partial response strategies.

We also faced challenges around performance, testing, and team adoption because reactive programming requires a different mindset. We improved this through code reviews, documentation, and reusable service patterns.

As a result, we reduced API response time, improved frontend load performance, and made the system more resilient during downstream failures.

A shorter version:

We converted a legacy synchronous application into a modern Angular and Spring Boot reactive application. The main goal was to reduce latency and improve reliability. We used RxJS operators like forkJoin, switchMap, and debounceTime on the frontend, and Spring WebClient with Resilience4j on the backend. The biggest challenges were identifying dependent vs independent API calls, handling partial failures, and helping the team adopt reactive patterns.



--------------------------------------------------------------------------------------------------
Give a real example from your project where you handled checked vs unchecked exceptions. Why did you choose that approach?

Here’s a strong real-project style answer:

In one of my Spring Boot microservices, we had an API that processed customer transaction data and also called downstream services and database operations. I handled checked exceptions and unchecked exceptions differently based on whether the situation was recoverable or business-related.

For example, while reading an uploaded file or calling an external service, we handled checked exceptions like IOException because these are external resource failures and the compiler expects us to handle them. In that case, I caught the exception, logged the details, and returned a meaningful error response to the user.

For business validation failures, such as customer not found, invalid transaction amount, or duplicate request, I used custom unchecked exceptions like CustomerNotFoundException or InvalidTransactionException. These extend RuntimeException because they represent business rule failures, and I wanted them to propagate to the global exception handler.

I used @RestControllerAdvice to centralize exception handling and return consistent API responses across the application.

Example:

try {
    fileService.processFile(file);
} catch (IOException ex) {
    log.error("File processing failed", ex);
    throw new FileProcessingException("Unable to process uploaded file");
}

Custom unchecked exception:

public class CustomerNotFoundException extends RuntimeException {
    public CustomerNotFoundException(String message) {
        super(message);
    }
}

Global handler:

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CustomerNotFoundException.class)
    public ResponseEntity<String> handleCustomerNotFound(CustomerNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }

    @ExceptionHandler(FileProcessingException.class)
    public ResponseEntity<String> handleFileProcessing(FileProcessingException ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(ex.getMessage());
    }
}

Interview summary:

I use checked exceptions for external or recoverable resource failures like file handling, database connectivity, or IO operations. I use unchecked exceptions for business validation failures and let them flow to a centralized global exception handler. This keeps service code clean and ensures consistent error responses.


If I have to share data between two Angular components, how do we do it?


You can share data between two Angular components in 4 common ways, depending on the relationship between components.

1. Parent to Child — @Input()

Use when parent sends data to child.

// child.component.ts
@Input() userName!: string;
<!-- parent.component.html -->
<app-child [userName]="name"></app-child>
2. Child to Parent — @Output()

Use when child sends data back to parent.

// child.component.ts
@Output() selectedUser = new EventEmitter<string>();

sendData() {
  this.selectedUser.emit('Suresh');
}
<!-- parent.component.html -->
<app-child (selectedUser)="receiveData($event)"></app-child>
3. Unrelated Components — Shared Service with BehaviorSubject

Use when components are not directly related.

@Injectable({
  providedIn: 'root'
})
export class SharedDataService {
  private messageSubject = new BehaviorSubject<string>('Default message');
  message$ = this.messageSubject.asObservable();

  updateMessage(message: string) {
    this.messageSubject.next(message);
  }
}

Component A:

this.sharedDataService.updateMessage('Data from Component A');

Component B:

this.sharedDataService.message$.subscribe(data => {
  this.message = data;
});

Best practice: use async pipe.

<p>{{ sharedDataService.message$ | async }}</p>
4. Route Parameters / Query Params

Use when sharing data through navigation.

this.router.navigate(['/details', userId]);

Read it:

this.route.paramMap.subscribe(params => {
  this.userId = params.get('id');
});
Interview Answer

If the components have a parent-child relationship, I use @Input() and @Output(). If they are unrelated components, I use a shared service with BehaviorSubject or another state management approach like NgRx. For route-based data, I use route parameters or query parameters.



=====================================================================================

Can you describe a complex front-end application you built using Angular, detailing your approach to component architecture and state management?


Here’s a strong interview-style answer you can use:

In one of my recent projects, I worked on a complex enterprise Angular application used to manage customer and operational workflows. The application had multiple modules, role-based screens, reusable forms, dashboards, search/filter functionality, and integration with several Spring Boot REST APIs.

From a component architecture perspective, I separated the application into feature modules and reusable shared components. For example, common UI elements like tables, search filters, buttons, confirmation dialogs, and form controls were placed in a shared module, while business-specific screens were organized under feature-level components. Each component had a clear responsibility, which made the application easier to maintain and test.

For API communication, I created dedicated Angular services that handled HTTP calls using HttpClient. Components did not directly manage API URLs or business logic; they consumed service methods and focused mainly on presentation and user interaction.

For state management, I used a combination of service-based state and RxJS. For feature-level shared data, I used BehaviorSubject to maintain state across components. For example, when a user updated search criteria or selected a record, the updated state was shared with related components through observables. In places where multiple API calls were required, I used operators like forkJoin, combineLatest, switchMap, debounceTime, and distinctUntilChanged.

To improve performance, I used lazy loading, trackBy for large lists, OnPush change detection where appropriate, async pipe to avoid manual subscriptions, and shareReplay to prevent duplicate API calls.

Overall, the architecture helped us keep the application scalable, maintainable, and performant while supporting complex business workflows.

Short version:

I built a complex Angular enterprise application with modular component architecture, reusable shared components, service-based API integration, and RxJS-based state management using BehaviorSubject. I optimized performance using lazy loading, OnPush, trackBy, async pipe, and RxJS operators like switchMap and forkJoin.


================================================================================

What are the differences between SQL and PL/SQL? When would you use each?


Basic Difference
SQL														PL/SQL
Query language									Procedural programming language
Used to interact with DB						Used to write business logic
Executes one statement at a time		Executes block of statements
Declarative										Procedural
No loops/conditions							Supports loops, conditions, exceptions
Standard language								Oracle extension of SQL

1. What is SQL?

SQL is used for:

CRUD operations

Meaning:

Create
Read
Update
Delete

Examples:

SELECT * FROM Employees;
INSERT INTO Employees VALUES (1, 'John');
UPDATE Employees
SET salary = 5000
WHERE emp_id = 1;
DELETE FROM Employees
WHERE emp_id = 1;

SQL focuses mainly on:

Data retrieval and manipulation


2. What is PL/SQL?

PL/SQL means:

Procedural Language extension to SQL

Mostly used in:

Oracle Corporation databases.

PL/SQL adds programming capabilities:

loops
conditions
variables
exception handling
procedures
functions
Example PL/SQL Block
DECLARE
    v_salary NUMBER := 5000;

BEGIN

    IF v_salary > 3000 THEN
        DBMS_OUTPUT.PUT_LINE('High Salary');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Low Salary');
    END IF;

END;
/

This is not possible in plain SQL.



Stored Procedure Example

PL/SQL is heavily used for stored procedures.

CREATE OR REPLACE PROCEDURE update_salary
(
   p_emp_id NUMBER,
   p_amount NUMBER
)
AS
BEGIN

   UPDATE Employees
   SET salary = salary + p_amount
   WHERE emp_id = p_emp_id;

END;
/
10. When Would You Use SQL?

Use SQL when:

Need to fetch/update data

Examples:

dashboard queries
reports
CRUD operations
joins
filtering
11. When Would You Use PL/SQL?

Use PL/SQL when:

Need business logic inside DB

Examples:

payroll calculation
banking transactions
batch jobs
auditing
complex validations
bulk data processing
Senior Interview Answer

SQL is a declarative language used for querying and manipulating relational data, while PL/SQL is Oracle’s procedural extension to SQL that supports variables, loops, conditions, exception handling, procedures, and functions. I use SQL for standard CRUD operations and reporting queries, while I use PL/SQL for complex business logic, batch processing, transaction handling, and stored procedures where processing closer to the database improves performance and consistency.


######################################################################################


Can you describe your experience with implementing RESTful web services in Java, including
any challenges you faced and how you overcame them?


I have implemented RESTful web services using Java and Spring Boot for enterprise applications involving customer, transaction, reporting, and integration workflows. Typically, I design APIs using a layered architecture: Controller, Service, Repository, and DTO layers. The controller handles HTTP requests, the service layer contains business logic, and the repository layer communicates with the database using JPA or JDBC.
I have worked with common REST methods like GET, POST, PUT, PATCH, and DELETE, and I usually design APIs around resources such as /customers, /orders, /payments, or /reports. I also use proper HTTP status codes, request validation, global exception handling, and consistent response structures.
One challenge I faced was handling multiple downstream service calls where one service was slow or unavailable. Initially, this caused delays and sometimes impacted the full response. To solve this, we implemented timeouts, retries, circuit breakers using Resilience4j, and fallback responses. For independent downstream calls, we used parallel execution with WebClient to reduce latency.
Another challenge was maintaining consistent error handling across multiple APIs. We solved this by using @RestControllerAdvice and custom exceptions, so every API returned a clean and predictable error response.
I also worked on performance improvements by optimizing SQL queries, adding caching, using pagination for large datasets, and reducing unnecessary payload fields in API responses.
Overall, my approach is to build REST APIs that are secure, scalable, well-documented, and easy for frontend teams like Angular developers to consume.

Short version:

I have built RESTful APIs in Java using Spring Boot with proper controller, service, repository, and DTO layers. I handled challenges around downstream failures using Resilience4j, improved performance with caching and pagination, and standardized errors using global exception handling.



==========================================================================================

How do you ensure that your REST services are both performant and secure, and can you provide an example of a scenario where you had to make trade-offs between the two?


I ensure REST services are performant and secure by designing them with both scalability and protection in mind from the beginning. On the performance side, I use pagination, caching, optimized SQL queries, connection pooling, async or parallel service calls where appropriate, and proper timeout settings. I also monitor P95/P99 latency, throughput, error rates, and database response time.

On the security side, I use authentication and authorization through Spring Security and JWT, role-based access control, input validation, HTTPS, secure headers, rate limiting, audit logging, and protection against common risks like SQL injection and broken access control.

One real trade-off I faced was around caching user-specific API responses. Caching improved performance because the same dashboard data was requested frequently, but we had to be careful not to expose sensitive or stale user data. Instead of caching everything globally, we cached only non-sensitive reference data and used short TTL caching for user-specific data. We also included user context in the cache key to avoid cross-user data leakage.

Another trade-off was with JWT token validation. Validating tokens on every request adds some processing overhead, but skipping validation would create a security risk. So we kept validation on every request but optimized it by using efficient signing algorithms, proper token expiration, and stateless authentication.

Overall, my approach is to never compromise critical security for performance. I optimize performance in a controlled way using caching, indexing, async calls, and payload reduction while keeping authentication, authorization, and audit controls intact.

Short interview version:

I improve REST performance using caching, pagination, optimized queries, async calls, connection pooling, and monitoring. I secure REST APIs using JWT, Spring Security, RBAC, input validation, HTTPS, rate limiting, and audit logging. A common trade-off is caching: caching improves performance but can expose stale or sensitive data, so I cache only safe data, use TTL, and include user context in cache keys.


======================================================================================

What are the
main features of Java that make it suitable for enterprise applications?


Java became the standard for enterprise applications because it provides reliability, scalability, security, portability, and a very mature ecosystem.

Main Features of Java for Enterprise Applications
1. Platform Independence

Java follows:

Write Once, Run Anywhere
5

How?

Java code
   ↓
Bytecode (.class)
   ↓
JVM
   ↓
Runs on Windows/Linux/Mac

Why important in enterprise?

Example:

Develop on Windows
Deploy on Linux servers

No code changes required.

2. Object-Oriented Programming

Java supports:

Encapsulation
Inheritance
Polymorphism
Abstraction

Example:

public class PaymentService {
}

Why important?

Enterprise applications need:

Reusable
Maintainable
Modular code

Example:

CustomerService
OrderService
PaymentService
3. Robust Exception Handling

Java provides:

checked exceptions
unchecked exceptions
try-catch-finally
try-with-resources

Why important?

Enterprise systems cannot crash due to one failure.

Example:

File failure
DB failure
Network failure

Application continues gracefully.

4. Automatic Memory Management

Java uses:

Garbage Collection
5

JVM automatically cleans unused objects.

Why important?

Prevents:

memory leaks
manual memory errors
5. Multithreading and Concurrency

Java supports:

Threads
Executor framework
CompletableFuture
Reactive programming

Why important?

Enterprise applications handle:

Thousands of users
Multiple transactions
Parallel processing

Example:

Payment processing
Notifications
Report generation

simultaneously.

6. Security

Java has built-in security features.

Examples:

bytecode verification
class loader isolation
cryptography APIs
secure networking

Why important?

Used in:

banking
healthcare
government systems
7. Rich Standard Libraries

Java provides APIs for:

collections
networking
concurrency
file handling
security
XML/JSON processing

Example:

List<String> users = new ArrayList<>();

Reduces development time.

8. JVM Performance

Modern JVM provides:

JIT compilation
optimized memory management
advanced garbage collectors

Examples:

G1GC
ZGC
Shenandoah

Why important?

Handles:

Large-scale enterprise workloads
9. Database Integration

Java integrates easily with databases using:

JDBC
JPA
Hibernate

Example:

Hibernate

Why important?

Enterprise apps are data-driven.

Examples:

customer data
transactions
reporting
10. Large Ecosystem

Java ecosystem includes:

Spring Boot
Apache Kafka
Apache Maven
Jenkins
Kubernetes

Why important?

Enterprise development needs:

Microservices
Cloud
CI/CD
Messaging
Monitoring
11. Scalability

Java supports:

monoliths
microservices
distributed systems

Examples:

Spring Boot + Kafka + Kubernetes

Can scale from:

100 users → millions of users
12. Backward Compatibility

Java values compatibility.

Example:

Older code can still run on newer JVMs

Important in enterprise where systems live for years.

Real Project Example

In one of my projects:

Angular
   ↓
Spring Boot microservices
   ↓
Kafka
   ↓
Oracle
   ↓
AWS EKS

Why Java?

Because we needed:

High concurrency
Security
Scalability
Fault tolerance
Long-term support
Senior Interview Answer

Java is well suited for enterprise applications because of its platform independence, object-oriented design, strong exception handling, automatic memory management, multithreading support, built-in security, mature ecosystem, and JVM performance optimizations. It integrates well with enterprise frameworks like Spring Boot, messaging systems like Kafka, databases, cloud platforms, and container orchestration platforms. In my projects, these features helped us build scalable, secure, and high-performance microservices that handle millions of transactions reliably.

Very Common Follow-Up

Why Java instead of Node.js or Python for enterprise systems?

Strong answer:

Java provides stronger type safety, mature multithreading, JVM optimizations, excellent tooling, long-term backward compatibility, and a proven enterprise ecosystem, which makes it ideal for large-scale, mission-critical systems.