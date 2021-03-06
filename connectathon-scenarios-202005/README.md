## Argonaut Subscriptions: May 2020 Connectathon Scenarios

We'll be participating in the Subscription track of the May 2020
Connectathon, testing the draft changes to the Subscriptions framework.  
The track is designed for clients and servers that want to try out a full
specturm of integration from:

1. [Patient Encounter notifications to a consumer app via REST-Hook](#scenario-1).
2. [Group Encounter notifications to an intermediary or clearing house via REST-Hook](#scenario-2).
- ? Patient Encounter notifications to an app via Websockets.
- ? Patient Encounter notifications to an app via Server Side Events (SSE).
- ? Define a new subscription topic for change notifications to a resource type.
- ? Work with authentication
    - REST-Hook
    - Websockets
- ? Multiple Notifications in a single bundle
- ? Explore graphs of resources on Subscriptions or Subscription Topics


For clarity in this document, the following words are defined as follows:

- Client: System or device **receiving** notifications.
- Server: System or device **sending** notifications.
- Trigger: System or device generating messages which **cause** notifications.

Each scenario will include three tiers of components.  Specifically, the scenarios use:
- Server Proxy: [GitHub](https://github.com/microsoft-healthcare-madison/argonaut-subscription-server-proxy)

    This is a server system (.Net Core CLI with Kestrel Http Server) which sits in front
    of a FHIR R4 Server and intercepts all calls relating to the scenarios. It handles 
    processing events and sending notifications via the specified channels to clients. 
    This was done to highlight the areas needed to implement the Server portion of Subscription handling.

    An implementer may choose to take a similar approach (of proxying calls), or may
    use a modified server directly.  Either option should result in the same behavior.

- Client UI: [GitHub](https://github.com/microsoft-healthcare-madison/argonaut-subscription-client-ui)

    This is a user interface project (React-based Web Application) which walks a user
    through the scenarios.  It includes everything that a client application would need
    to maintain subscriptions.

    Because a Web Application cannot host its own endpoints, while WebSocket scenarios
    interact directly with the Server, REST endpoints are hosted in a Client Host project
    and communication between the UI and Host is handled via WebSockets.


- Client Host: [GitHub](https://github.com/microsoft-healthcare-madison/argonaut-subscription-client)

    This is a server system (.Net Core CLI with Kestrel Http Server) which:
    - hosts the UI (if desired)
    - manages REST endpoints
    - communicates received REST notifications to the Client UI
    - Triggers events on the server (e.g., sends Encounter resources).

    Generally, this project is an artefact of testing.  However, implementers may be 
    interested in details about how the UI interacts with notifications or how REST endpoints
    are managed.


## <a name='scenario-1'>Scenario 1</a>: Single-Patient Encounter notifications via REST-Hook (e.g. to a consumer app)

#### Subscription information

- SubscriptionTopic: [`encounter-start`](https://github.com/argonautproject/subscriptions/blob/master/canonical/subscriptiontopic-encounter-start.json)
- Allowed filter searchParamNames: `patient`
- Allwoed filter searchModifiers: `=`
- Allowed channelTypes: `rest-hook`
- Allowed contents: `empty`, `id-only`

#### Workflow - [Diagram](https://github.com/argonautproject/subscriptions/blob/master/connectathon-scenarios-202005/svg/Scenario1_workflow.svg) - [Simplified Graph](https://github.com/argonautproject/subscriptions/blob/master/connectathon-scenarios-202005/svg/Scenario1_graph.svg)

- Optional SubscriptionTopic Discovery:
    - Client requests all `SubscriptionTopic` resources from the server.
    - Server responds with a bundle of `SubscriptionTopic` resources.
1. Select or Create a Patient:
    - Selecting an existing Patient:
        1. Client asks User for Search Information related to a Patient or information to Create a Patient.
        2. Client asks Server for a list of matching patients.
        3. User selects a patient to use in the Scenario.
    - Creating a Patient:
        1. Client asks Users for Patient Information.
        2. Client Creates Patient record on the Server.
2. Client creates a REST endpoint to receive notifications on.
3. Client sends [Subscription](http://build.fhir.org/subscription-admission.html) request to Server.
4. Server responds with status: `requested`.
5. Server sends a [Handshake bundle](http://build.fhir.org/notification-handshake.html) to the REST endpoint.
6. If successful, Server updates the Subscription to status: `active`.
7. Trigger Server sends Encounter for selected patient.
8. Server sends event notification ([empty](http://build.fhir.org/notification-empty.html), [id-only](http://build.fhir.org/notification-empty.html)) to client REST endpoint.
9. Client Notifies user that a notification has been received.

#### Client Information

- (Optional) Implement SubscriptionTopic discovery
- Either allow searching/selecting or creating a Patient
- Create a REST endpoint
- Send a Subscription request to the server
- Handle Subscription REST Handshake
- Receive notifications via REST endpoint
- Notify user that an event has occurred.

#### Server Information

- Implement SubscriptionTopic discovery
- Manage Subscriptions
  - Accept Subscription requests
  - Validate REST-Hook channel endpoints
- Have some way of triggering events (e.g., accepting Encounter resources)
- Filter and match for applicable Subscriptions
- Send event notifications

#### Trigger Information

- Accept a patient id filter parameter
- Generate a Patient record (if it does not exist)
- Generate an Ecounter record for the specified patient
  - Once
  - Periodic


## <a name='scenario-2'>Scenario 2</a>: Group Encounter notifications to an intermediary or clearing house via REST-Hook

#### Subscription Information

- SubscriptionTopic: [`encounter-start`](https://github.com/argonautproject/subscriptions/blob/master/canonical/subscriptiontopic-encounter-start.json)
- Allowed filter searchParamNames: `patient`
- Allwoed filter searchModifiers: `in`
- Allowed channelTypes: `rest-hook`
- Allowed contents: `empty`, `id-only`


#### Workflow [(Diagram)](https://github.com/argonautproject/subscriptions/blob/master/IntermediaryWorkflows/svg/GenericGroup.svg)


#### Client Information

- (Optional) Implement SubscriptionTopic discovery
- Determine filter for Encounter based on Group Membership
- Create a REST endpoint
- Send a Subscription request to the server
- Handle Subscription REST Handshake
- Receive notifications via REST endpoint
- Notify user that an event has occurred.

#### Intermediary Information

- Implement SubscriptionTopic discovery
- Manage Subscriptions
  - Accept Subscription requests
  - Validate REST-Hook channel endpoints
- Create REST endpoint for Server events
- Subscribe to Server for events
- Listen for event notifications
- Filter and match for applicable Subscriptions
- Send event notifications to Client

#### Server Information

- Implement SubscriptionTopic discovery
- Manage Subscriptions
  - Accept Subscription requests
  - Validate REST-Hook channel endpoints
- Accept Encounter resources
- Accept Group Membership resources/changes
- Filter and match for applicable Subscriptions
- Send event notifications

#### Trigger Information

- Accept patient group membership information
- Generate one or more Patient records for a specified group
- Generate one or more Ecounter records for a relevant patient
  - Once
  - Periodic

## Potential Scenario: Patient Encounter notifications to an app via Websockets

#### Subscription Information

- SubscriptionTopic: [`encounter-start`](https://github.com/argonautproject/subscriptions/blob/master/canonical/subscriptiontopic-encounter-start.json)
- Allowed filter searchParamNames: `patient`
- Allwoed filter searchModifiers: `=`, `in`
- Allowed channelTypes: `websocket`
- Allowed contents: `empty`, `id-only`, `full-resource`

## Potential Scenario: Patient Encounter notifications to an app via Server Side Events (SSE)

#### Subscription Information

- SubscriptionTopic: [`encounter-start`](https://github.com/argonautproject/subscriptions/blob/master/canonical/subscriptiontopic-encounter-start.json)
- Allowed filter searchParamNames: `patient`
- Allwoed filter searchModifiers: `=`, `in`
- Allowed channelTypes: `server-side-event`
- Allowed contents: `empty`, `id-only`, `full-resource`


## Potential Scenario: Define a new subscription topic for change notifications to a resource type

#### Basic Information

- SubscriptionTopic: TBD
- Allowed filters: 

## Potential Scenario: Work with authentication (REST-Hook)

#### Subscription Information

- SubscriptionTopic: [`encounter-start`](https://github.com/argonautproject/subscriptions/blob/master/canonical/subscriptiontopic-encounter-start.json)
- Allowed filter searchParamNames: `patient`
- Allwoed filter searchModifiers: `=`
- Allowed channelTypes: `rest-hook`
- Allowed contents: `empty`, `id-only`, `full-resource`


## Potential Scenario: Work with authentication (Websocket)

- SubscriptionTopic: [`encounter-start`](https://github.com/argonautproject/subscriptions/blob/master/canonical/subscriptiontopic-encounter-start.json)
- Allowed filter searchParamNames: `patient`
- Allwoed filter searchModifiers: `=`
- Allowed channelTypes: `websocket`
- Allowed contents: `empty`, `id-only`, `full-resource`
