![Image of compontents](https://github.com/alidabour/billing_system_design/design.png)

- **Payment Service** listen on **Payment Requests Topic** on receiving a message it will fetch user credit card info from its DB and prepare and publish a message on **Payment   Gateway Topic**.
- **Subscription Service** store user subscriptionsdata. Will have a daily  job to start payment process by publishing message to **Payment Requests Topic** .
- **Payment Gateway A, B, ... Service** listen on **Payment Gateway A, B, ... Topic** And start a payment process with a Payment Provider. After processing it will publish a message on **Response Topic**
- **Invoice Service** listen on **Response Topic** on receiving a successful message it will create a digital invoice (PDF file) and save it on a **Storage Bucket** and send an email and SMS.
- **Payment Service** listen on **Response Topic** and update DB with the request status, In case of unsuccessful request it  notify the user to fix his payment method.
- **Payment Service** store every transaction and can supply user with payment history.
- **Notifier Service** can supply the user with digital invoices.
- **Activator Service** listen on **Response Topic** and store if a user can't access Company products.

**Save and Validate Credit Card**
```mermaid
sequenceDiagram
Client -->> Payment Service: Save my credit card
Payment Service ->> Client: Process Started.
Payment Service -->> Validate CC. A Topic: Validate Credit Card.
Validate CC. A Topic -->> Payment Gateway A: Validate Credit Card.
Payment Gateway A -->> Payment Provider A: Validate Credit Card.
Payment Provider A -->> Payment Gateway A: Credit Card is valid.
Payment Gateway A -->> Credit Card Response Topic: Credit Card is valid.
Credit Card Response Topic ->> Payment Service: Credit Card is valid.
Note right of Payment Service: Store credit card info. 
```
**Requesting Payment**
```mermaid
sequenceDiagram
Subscription Service ->> Payment Req. Topic: Charge User X for Z$
Note left of Subscription Service: Check DB for due<br> payment on a daily <br>basis.
Payment Req. Topic-->>Payment Service: Charge User X, for Z$
Company Products->>Payment Req. Topic:  Charge User X, for Z$
Payment Req. Topic-->>Payment Service: Charge User X, for Z$
```
**Processing Payment Request**
```mermaid
sequenceDiagram
Note left of Payment Service: Fetch User credit<br> card data for the DB
Payment Service-->>Payment Gateway A Topic: Charge Z$ from Credit Card XXXX
Note left of Payment Service: Store payment<br> request
Payment Gateway A Topic ->> Payment Gateway A: Charge Z$ from Credit Card XXXX
Payment Gateway A ->>Paymnet Provider A: Charge Z$ from Credit Card XXXX
```
**Successful Payment Request**
```mermaid
sequenceDiagram
 Paymnet Provider A-->>Payment Gateway A: Payment Successful
 Payment Gateway A -->> Response Topic: Payment Request ID Y for User X is successful.
Response Topic -->> Invoice Service: Payment Request ID Y for User X is successful.
Note right of Invoice Service: Invoice create a digital<br> invoice (PDF file) and<br> save it on a Storage <br>Bucket and send an <br>email and SMS.
Response Topic -->> Activator Service: Payment Request ID Y for User X is successful.
Note right of Activator Service: Remove User X from<br> the blocked list.
Response Topic -->> Payment Service: Payment Request ID Y for User X is unsuccessful.
Note right of Payment Service: Update request ID Y<br> with sucessful status.<br> Remove user from <br> "reminder" email list.
```
**Unsuccessful Payment Request**
```mermaid
sequenceDiagram
 Paymnet Provider A-->>Payment Gateway A: Payment unsuccessful
 Payment Gateway A -->> Response Topic: Payment Request ID Y for User X is unsuccessful.
Response Topic -->> Invoice Service: Payment Request ID Y for User X is unsuccessful.
Note right of Invoice Service: Invoice send an <br>email and SMS for the<br> unsuccessful payment.
Response Topic -->> Activator Service: Payment Request ID Y for User X is successful.
Note right of Activator Service: Add User X to the <br> blocked list.
Response Topic -->> Payment Service: Payment Request ID Y for User X is unsuccessful.
Note right of Payment Service: Update request ID Y<br> with unsuccessful status.<br> Add user to "reminder"<br> email list to be <br> reminder every <br>X period to fix his<br> payment method
```

**Manual Payment**
```mermaid
sequenceDiagram
Client -->> Payment Service: GET request to fetch unsuccessful payment request
Payment Service ->> Client: required information to repay.
Client -->> Payment Service: POST request to repay.
Payment Service ->> Client: Return  Payment started msg.
```

**Billing History**
```mermaid
sequenceDiagram
Client -->> Payment Service: GET request to fetch payment history
Payment Service ->> Client: Payment history
Client -->> Invoice Service: Download invoice for request ID X.
Note right of Invoice Service:  Check the required<br> file on Storage Bucket.
Invoice Service->>Client: Redirect to the file URL 
```

**Company Products ask for active user**
```mermaid
sequenceDiagram
Company Products -->> Activator Service: Is user X blocked.
Note right of Activator Service: Check DB if user exist.
Activator Service ->> Company Products: Yes he is.
```

**Technologies**
-  Services could use Go for its speed and Cloud features.
- Messaging Service could use Google Pub/Sub for easy of setup and uses. That would introduce the necessity of handling [duplicate messages](https://cloud.google.com/pubsub/docs/faq#duplicates) and [messages order](https://cloud.google.com/pubsub/docs/faq#order). Another option is to used Kafka but that will introduce complexity compared to Google Pub/Sub.
- DB could be a NoSQL database as Google Datastore for the benefit of horizontal scalability.

**Notes**
- User will get a token at login and it will be used for REST APIs authentication. 
- Every service will store in its DB the minimal User representation required by the service to operate.
- Every service will build upon a **Service Template** where we keep shared logic sync as Token Verification, Logging and Monitoring.