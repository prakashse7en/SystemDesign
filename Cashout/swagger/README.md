#Auto Fund Out Feature

**Feature:** Auto Fund out  
**System:** Payme for business  
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
* **Monitoring and observability** - Dashboards to be created to monitor auto fund out performance. Proper Alerting to be created and report Incident management team via x-matters in case of any issues.

## Design Deep dive

### Frequency Types: Hourly, Weekly, Daily

| Frequency Type | Attributes | Comments |
|----------------|------------|----------|
| **Hourly** | freqType: "H"<br>Hour(0-23) - default 10pm<br>minimumBalance: default 0 | Day is not required here |
| **Weekly** | freqType: "W"<br>Day(0-6) - default 6<br>Hour(0-23) - default 10pm<br>minimumBalance: default 0 |  |
| **Daily** | freqType: "D"<br>Hour(0-23) - default 10pm<br>minimumBalance: default 0 | Day is not required here |
