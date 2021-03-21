# Engineering Manager - System design exercise

This is the answer for the system design exercise from ACME's API, where our goal is to calculate the credit score of a user based on their past financial activity.
According to the exercise, there are some requirements we need to fulfill:
<ul>
<li>Users will authorize ACME to access their banking data with an oauth2-style flow, passing a token to ACME</li>
<li>Calls to the Bank API through HTTP</li>
<li>Data schould be anonymized and stored in a database</li>
<li>The final results of the credit score calculation will be sent with a webhook to the client once ready</li>
<li>Clients can also decide to force a synchronous operation, in which case they will leave a connection open to ACME until both the data retrieval and data crunching is complete</li>
<li>This service has to be able to satisfy a 10 credit score requests/second throughput</li>
</ul>

Besides requirements, there are some premisses to consider:
<ul>
<li>The credit score calculation is a CPU-bound operation that performas number crunching and typically takes several seconds to complete (consider 10 seconds)</li>
<li>Each Bank API call takes several seconds to complete (consider 10 seconds)</li>
</ul>

## ACME API

The following Sequence Diagram can help us understand how messages would be exchanged among each actor within our system:

![Sequence Diagram](/UseCase.png "Sequence Diagram")

We will use a microsservices architecture, where each service should be stateless, autonomous and provide a single functionality. Services will exchange messages through an event queue.
We should split our system into 5 services:

### ACME API REQUEST
This will be the service responsible to receive requests from our partners, provide the oauth2 interface to get the token, and to build the bank api call requests.

### ACME API RESPONSE
This service should listen to the event queue, waiting for credit scores messages in order to call our partner's webhook to send our final result.

### BANK API REQUEST
This service should listen to the event queue, waiting for Acme API requests in order to call Bank's API to get user's financial activity.

### CREDIT SCORE CALCULATION
This service should receive requests for credit score, calculate and provide a response with the resulted score.

### STORE SERVICE
This service should listen to the event queue, and for every Bank API response and every credit score Calculation response it should anonymize sensitive data and store in a database.

This is how our microsservices architecture should look like:

![Microsservices Architecture](/Architecture.png "Microsservices Architecture")

## TEAM STRUCTURE
As we planned to have 5 services, our first approach in order to organize our team may consider the Conway's law:

> <i>Any organization that designs a system  will produce a design whose structure is a copy of the organization's communication structure.</i>

So, we would at first consider 6 squads - what would create small but well scope-defined squads - one for each service, and a sixth team to care for our infrastructure, as a microsservices architecture is scalable but complex, and some team not focusing on any business rule should care and maintain our system fulltime.

Considering our proposed engineering team of 20 ICs and 4 product managers, we would divide the team as:
<li> 2 product managers will take care of 2 squads each, and our division should consider length of scope and communication with external teams</li>
<li> ACME API REQUEST and ACME API RESPONSE: each squad would have 3 ICs and the same product manager</li>
<li> BANK API: this squad would also have 3 ICs, but an exclusive product manager, as it will require several meetings and communications with external partners (the banks)</li>
<li> CREDIT SCORE CALCULATION: this squad would have 3 ICs and an exclusive product manager, due to its complexity and probable length of business rules to be consider into the calculation</li>
<li> STORE SERVICE: this squad would have 3 ICs and a product manager, to be split with the INFRA squad</li>
<li> INFRA STRUCTURE SQUAD: this would be the largest squad (5 ICs) as it would have to provide the architecture and scalability for the other squads. It may share its product manager with another squad because its main challenges should be technical, and not from business (what would need an extra effort from a PM)</li>

![Organization Chart](/OrgChart.png "Organization Chart")

## PROJECT MILESTONES

As we are considering a microsservice approach, the first and most important point to define is the contract among services. This is essential to let all squads work independently and in parallel, until our integration test phase.
The second milestone is to the infrastructure squad to provide an development environment, considering especially the services requirements, and the common parts, for instance the Event Queue.
The third milestone is the development of each individual service, where our Definition of Done should be have the service developed (according to the contracts prior defined) and individually tested (using mocked requests and responses).
The next milestone should consider the delivery of the QA environment and a CI/CD pipeline to build and deliver the services.
At this point, we should be ready to have integration tests, so the next milestone should consider the services tested as one system.
The next should be the CI/CD pipeline to the production environment.

![Milestones](/Milestones.png "Milestones")

## SCALABILITY
If we need to scale our system, for instance go from 10 credit score requests/second to 100 requests/second, we should consider it a matter of architecture in order to handle it - we would not make changes in our codebase (as we have the premisse to have credit scores calculation and bank api requests to last around 10 seconds each), we would keep the length of each call, but provide an architecture to have more requests in parallel.
In our API Gateway, we would attach a Load Balancer, and our cluster hosting our microsservices should also have the option of auto scaling. If we consider our architecture to be hosted on AWS, we can make use of ELB (Elastic Load Balancer) and Auto Scaling options on ECS (EC2 Container Service).
We also have other options, like working with Kubernetes in Rancher, which would also make the options available for auto scaling and load balancing.
