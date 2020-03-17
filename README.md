# Overview
This is a proposed solution design for a Flight Information Display system.
Total Estimate is 30 days to implement for 4 developers, 1 tester and 1 product owner.

## Requirements
- [x] Flight Scheduling - Admin Service can persist flight schedules
- [x] Flight Changes - Admin Service can persist changes
- [x] Flight Information Display - Info Service can display flight information
- [x] Flight Information API - Info Service provides HTTP endpoint for flight information
- [x] Public Flight Information Display - Info Website is exposed via a Public API Gateway
- [x] Scalable Flight Information Display - Info Website delivered via CDN
- [x] Flight Push Notifications - Basic idea is there, but requires the external service manage subscriptions
- [x] Flight Data Isolation - Basic implementation, but can be improved with stricter tenant database and application instance isolation.
- [x] Non-Public Flight Changes - Admin Service in Private Network accessible with VPN

## Trade-offs
- Getting data from Admin to Display - Polling vs Event Based - Event based increases complexity but improves decoupling and performance
- Single vs Multiple APIs - Multiple APIs increases complexity, but improves decoupling, security diversity, performance
- Databases - MongoDB vs SQL - MongoDB loses some of the traditional comforts of SQL and it's tools, but we get amazing performance and scaling.
- Data Storage - State vs Event Sourcing - Event Sourcing allows us to scale up performance massively while maintaining consistency and transition over to an event based system.

## Assumptions
* Cloud environment for deployment
* Microsoft oriented stack is familiar or preferred
* Caching is not yet needed and can be introduced at a later stage

## Had no time for:
* Infrastructure components
* Events, their details or flows
* Identity Provider and security
* Network details
* Service Level details

# Components
![alt text](docs/diagrams-components-4.png "Components")

#### Common Tech
* `Docker` - For development and operational benefits.
* `OpenTelemetry -> Application Insights` - Fully managed telemetry for a good price. Can easily be swapped out of course!

#### Common Infrastructure
* `Private VNet` - All components are deployed to private networks to secure them.

## **Event Store**
This is where we store all of our events that are produced by Domain logic commands.
### Tech
* `NATS Streaming Server` -  High scale message broker and event store that can handle and persist all the events we can throw at it.

## **Flight Info DB**
We store all of the flight information ready for read-only display here. Old data automatically gets deleted.

### Tech
* `MongoDB -> CosmosDB` - We need a datastore that provides speed, scalability and TTL for data records.

### Models
**Flight Departure Information** - This collection contains documents that can be used by the API as-is and should not require additional processing before serving it in an HTTP request.
```JSON
[
    {
        "Airline": "",
        "FlightNumber": "",
        "Destination": "",
        "ScheduledDepartureDateTime": "",
        "EstimatedDepartureDateTime": "",
        "ActualDepartureDateTime": "",
        "Status": "",
        "DepartureGate": "",
        "ExpireAt": ""
    }
]
```
**Flight Departure Status** - This collection contains a rolling snapshot of flight departure schedules and their current status. This data is used to project the optimized documents for the **Flight Departure Information**.
```JSON
{
    "Airline": "",
    "FlightNumber": "",
    "Destination": "",
    "ScheduledDepartureDateTime": "",
    "EstimatedDepartureDateTime": "",
    "ActualDepartureDateTime": "",
    "Status": "",
    "DepartureGate": "",
    "LastUpdateDateTime": "",
    "ExpireAt": ""
}
```

## **Flight Admin DB**
We store all of the flight schedules and aggregated flight information. The collections have a **airline** field that is used for basic tenant isolation. Tenant validation occurs before writing the data. The collections can be split up so that each **tenant/airline** as it's own set of collections to further increase tenant isolation. This comes at the cost of greater complexity and costs.

### Tech
* `MongoDB -> CosmosDB` - We need a datastore that provides speed & scalability.

### Models
**Flight Departure Schedules** - Contains the schedules for flight departures.
```JSON
{
    "Airline": "",
    "FlightNumber": "",
    "Destination": "",
    "DepartureTime": "",
    "DepartureDays": ["","",""],
    "CreatedAt": "",
    "UpdatedAt": ""
}
```
**Flight Departure Status** - This collection contains flight statuses based on schedules and events.
```JSON
{
    "Airline": "",
    "FlightNumber": "",
    "Destination": "",
    "ScheduledDepartureDateTime": "",
    "EstimatedDepartureDateTime": "",
    "ActualDepartureDateTime": "",
    "Status": "",
    "DepartureGate": "",
    "CreatedAt": "",
    "UpdatedAt": "",
    "ExpireAt": ""
}
```

## **Flight Info Display**
The Flight Info Display is an internet facing website that displays the upcoming flight departures. It communicates with the Flight Info API over HTTP using JSON serialization.

### Tech
* `HTML & Javascript` - The website needs to be lightweight and fast to render.

### Infrastructure
* `Public API Gateway` - Exposes the website to the internet
* `CDN` - Offload edge traffic to a content delivery network to optimize website performance and availability

### SLO's
* Availability - `>99.99% of requests 2XX`
* Performance - `>95% of requests <200ms`

## **Flight Info API**
The Flight Info API is an internet facing HTTP API that provides flight departure information.

### Tech
* `ASP.Net Core Web API` - We need some basic HTTP API functionality.

### Infrastructure
* `Public API Gateway` - Exposes the API to the internet

### SLO's
* Availability - `>99.99% of requests 2XX`
* Performance - `>95% of requests <50ms`

## **Flight Info Aggregator**
The Flight Info Aggregator updates the state in **Flight Status DB** based on events produced by the **Flight Admin Service**.

### Tech
* `.Net Core` - Enough to build an event processing application

### SLO's
* Availability - `>99.999% events processed successfully`
* Performance - `>99% of events should process in less than <50ms`

## **Flight Admin UI**
The Flight Admin UI is a privately accessed website that allows operators to login and manage flight schedules and statuses. It communicates with the Flight Admin API over HTTP using JSON serialization.

### Tech
* `VueJS` - The website needs crud and other basic UI support, but still remain light weight.

### Infrastructure
* `Private API Gateway` - Exposes the website to users with access to the Private VNet

### SLO's
* Availability - `>99.9% of requests 2XX`
* Performance - `>95% of requests <500ms`

## **Flight Admin API**
The Flight Admin API provides the operations needed for flight scheduling and management.

### Tech
* `ASP.Net Core Web API` - We need an HTTP API framework/library/language that can provide a rich toolset for building domain logic.

### Events
// TODO

### Security
The API expects an Authorization header with an **Access Token(JWT)** for each request. This Token will determine what data a user can access and what operations they can perform.

### Infrastructure
* `Private API Gateway` - Exposes the website to users with access to the Private VNet

### SLO's
* Availability - `>99.99% of requests 2XX`
* Performance - `>95% of requests <50ms`

## **Notification Integration - Integration**
The Notification Integration calls the Notification Provider whenever there is a flight status event.

### Tech
* `.Net Core` - Enough to build an event processing application

### SLO's
* Availability - `>99.999% events processed successfully`
* Performance - `>99% of events should process in less than <50ms`

# General
## Network
![alt text](docs/diagrams-network.png "Components")

## Testing
All levels of the testing pyramid should be implemented for Application Code and Infrastructure Code:
* Static analysis and unit tests should be run with every push to source control
* Unit test coverage should be above 95%, excluding plumbing code
* Integration and end-to-end tests should run before a pull request is merged to master

## Deployment
Development should follow the [GitHub Flow](https://guides.github.com/introduction/flow/)(not GitFlow!) workflow and store all code in GitHub.
All deployments should be fully automated with these tools:
* CI/CD - GitHub Actions - Very easy, fast & secure
* Infrastructure - Terraform - Best way to declaratively code and deploy infrastructure
* Applications - Docker - Run the same application environment locally and in production with ease

# Estimation
There is a pretty equal distribution of work in all areas of development(Frontend, Backend, Infrastructure, Automation).

## Team
The team allows pairing options, skills overlap, and flexibility:
* FrontEnd Developer - 2
* Backend Developer - 2
* Tester - 1
* Product Owner - 1

## Numbers
|Component| Days |
|-|-|
|Setup & Automation|5|
|Flight Admin|10|
|Flight Info|10|
|Notification|5|
|||
|**Total**|**30**|
|||