# Auto Fund Out Feature

**Feature:** Auto Fund out  
**System:** Fintech  Merchant App  
**Stake holder:** Business Analyst  
**Version:** V1.0  
**Approval:** Senior Solution Architect  

## Problem Statement

Existing system merchant can do manual cashout feature as most of the merchant don't actively login and have to trigger cashout manually instead we need to come up with a solution such there is automatic way to cashout (leveraging existing logic of cashout)

## Functional Requirements

* **Auto Fund out** - user can able to schedule the fund out like daily, weekly, hourly. Minimum balance to be captured for the merchant if not provided to be default to 0 which is the amount to be left when performing auto fund out.
* **Merchant flexibility** - Merchant to be provided flexibility to update auto fund out status inactive/active and can able to see the same in app (IOS /AOS)
* **System compatibility** - Existing manual cashout to be undisturbed and no performance degradation to be introduced in transaction ms by adding this feature

## Capacity Estimation

Existing manual cashout api calls is ~<2 TPS as merchant don't do cashout every day.  
As per current situation total number of merchant - 40000. Worst case 40000 merchants choose to have same frequency and trigger cashout.  
We can use event driven design rather than synchronous calls to support auto fund due to systems growing nature of merchants. Event hubs partitions increased to handle high traffic

## Non Functional Requirements

* **Security** - Authentication and Authorization (RBAC) flow to initialized as scheduled job. Use existing Internal JWT token(role is internal) to trigger auto fund out job
* **Scalability** - Since it is event driven design there shall be spike at times when traffic is high. Indexing to be in place for queries to be added. There shall be spike at certain times ensure correct set of partitions present for event hubs.
* **Monitoring Alerting and observability** - Dashboards to be created to monitor auto fund out performance. Proper Alerting to be created and raise alerts incase of any issues during the auto fundout flow

## Design Deep dive

### Frequency Types: Hourly, Weekly, Daily

| Frequency Type | Attributes                                                              | Comments |
|----------------|-------------------------------------------------------------------------|-------------
| **Hourly**     | freqType: "H"<br>Hour(0-23) - default 10pm<br>minimumBalance: default 0 | Day is not required here |
| **Weekly**     | freqType: "W"<br>Day(0-6) - default 6<br>Hour(0-23) - default 10pm<br>  |minimumBalance: default 0 |                 
| **Daily**      | freqType: "D"<br>Hour(0-23) - default 10pm<br>minimumBalance: default 0 | Day is not required here |

## Existing logic 
<img width="391" height="471" alt="image" src="https://github.com/user-attachments/assets/b63b5e15-b462-40de-a2fd-0fdccbfde34c" />

* 1 and 2 User fetches IDToken from Azure ADB2C and gets Operation token after verifying the pin [ operation token valid for one operation ] . 
* 3 and 4 Cashout is triggered which internally calls ledger to do the money movement
* 5 Based on ledger status for cashout operation success or failure notifications are sent . Status of Uncompleted cashout will be known during reconciliation and corresponding notifications are sent  accordingly.

## AUTO Fund out logic
<img width="509" height="191" alt="image" src="https://github.com/user-attachments/assets/eac67ed0-2ba6-4197-84f3-38ae52e93b3b" />

Auto Fund Out creation /read and update
* 1.Get ADB2C IDToken and set /view/update auto fundout . Corresponding auto fundout details to be persisted  in below table 

Table name - Orgn_cust_auto_fund_out

| Column Details |
|----------------|
| **Column Name:** Orgn_cust_auto_fund_out_id<br>**Data type:** binary |
| **Column Name:** Orgn_cust_merch_id<br>**Data type:** binary |
| **Column Name:** Freq_type<br>**Data type:** varchar(10) |
| **Column Name:** isEnabled<br>**Data type:** byte |
| **Column Name:** day<br>**Data type:** tinyint |
| **Column Name:** hour<br>**Data type:** tinyint |
| **Column Name:** minimum_bal<br>**Data type:** decimal(10,3) |

## Auto FundOut Batch job

<img width="639" height="661" alt="image" src="https://github.com/user-attachments/assets/b54c2b13-37f0-4302-8155-2b8468b9b371" />

* 1.1.1 cron job configured in scheduler microservices to run hourly auto fundout job 
* 1.1.2 Invokes Business profile microservices to pick if any active auto fundout present at this time . If yes then send an event to Auto fundout event producer
* 1.2.1 and 1.2.2 Auto fundout event consumer consumes the event in payment transaction ms and performs cashout invoking ledger 
* 1.2.3 Based on ledger status for cashout operation success or failure notifications events are sent . Status of Uncompleted cashout will be known during reconciliation and corresponding notifications are sent  as events to merchant_email_notification_event producer accordingly.
* 1.3.1 merchant_email_notification_event consumer consumes the event in notification ms and sends the email notification and audit of this notification is persisted into table

## Swagger 
https://github.com/prakashse7en/SystemDesign/blob/main/Cashout/swagger/autofundout.yaml

| Endpoint                                             | Method | Description |
|------------------------------------------------------|--------|-------------|
| /business-profiles/business/{businessId}/autofundout | POST | Create auto fundout configuration |
| /business-profiles/business/{businessId}/autofundout | GET | Get auto fundout configuration |
| /business-profiles/business/{businessId}/autofundout | PUT | Update auto fundout configuration |
| /business/autofundout                                | POST | Trigger batch processing |


## Business Rules & Validations

### 1. Frequency Type Rules:
* **H (Hourly):** Executes every hour at specified minute (typically :00)
* **D (Daily):** Executes every day at specified hour
* **W (Weekly):** Executes on specified day of week at specified hour

### 2. Day Field Requirements:
* **Hourly:** Day field not required
* **Daily:** Day field not required
* **Weekly:** Day field required (0-6, where 0=Sunday, 6=Saturday)

### 3. Default Values:
* **Hour:** 22 (10:00 PM) if not specified
* **Day:** 6 (Saturday) for weekly frequency

### 4. Default Values:
* **Minimum Balance:** 0 if not specified
* **Enabled Status:** false by default

### 5. Validation Rules:
* **Hour Validation:** Must be in 24-hour format (00-23)
* **Day Validation:** 0-6 for weekly, ignored for daily/hourly
* **Balance Validation:** Must be >= 0
* **Frequency Validation:** Only H, D, W allowed

### 6. Business Logic :
* Only one configuration per business
* Disabled configurations stored but not executed

## Performance Benchmarks
| Endpoint | Expected Response Time | Max Acceptable |
|----------|----------------------|----------------|
| POST /autofundout | < 100ms | < 500ms |
| GET /autofundout | < 100ms | < 300ms |
| PUT /autofundout | < 100ms | < 500ms |
| POST /business/autofundout (batch) | < 100ms | < 200ms |


## Queries and Decision
Using existing synchronous call vs event driven design -> event driven design is better and its not blocking in nature .
Agreed to use event driven design as it is non blocking compared to blocking synchronous calls

## Future Enhancements
Include frequency Monthly to handle  leap year /non leap year dates
