## Introduction

The Encounters Implementation Guide is based on FHIR Version R5 and defines the minimum conformance requirements for supporting patient Encounter notifications.

As of this writing, the Subscriptions R5 framework is still under active development.  The guidance provided in this Guide is considered substantially complete, with the understanding that the canonical instances of resources will change in alignment with R5.  When FHIR R5 is published, an update to this Guide will be published updating this note.

Last updated from build.fhir.org (R5 development branch) on February 06, 2020.

## Background

In 2019, the FHIR Infrastructure WG decided that the existing `Subscription` mechanism did not work for all required use cases, and that it would need to be redesigned.  A resource redesign with breaking changes is not something the FHIR community undertakes lightly.  While the Subscriptions mechanism available in R4 worked for some use cases, there were implementation issues which were insurmountable in others.  The changes in R5 can be broken down into a few key points:

  - Retain functionality available in R4
  - Split `Subscription` into two definition resources: `Subscription` and `SubscriptionTopic`
  - Allow servers to explicitly support specific `SubscriptionTopics`
  - Simplify filter language (base on existing parts of FHIR)
  - Specify filter restrictions in `SubscriptionTopic` (canFilterBy) to ease implementation

A longer discussion about the redesign can be found [here](#more-on-subscriptions-r5).

## Rationale

- Why Encounter Notifications?

  - Encounter notifications are currently quite relevant in the U.S., given forthcoming rules and regulations (e.g., both patient access and care team notifications).  

  - Encounters are relatively simple to conceptualize

  - Encounters are well modeled and defined

  - Encounter resources are well supported in current FHIR implementations

  - Encounter notifications enable many workflows, both internal and external to a provider

  - Low political ramifications

- Why an Argonaut Guide?

  - The Argonaut Project felt that establishing canonical resource instances for Encounters would allow for faster and broader adoption.

  - By engaging vendors and providers in a tight feedback loop, the Argonaut Project could help move the R5 redesign work forward.


## Use case: encounter notifications

There are a few key use cases that Argonaut focused on in terms of Encounter notifications:

- Single patient (identified by id) start of encounter

  This use case covers a system being notified when a specific patient of interest has started a new encounter.

  For example, a consumer device (e.g., phone) receiving notifications that a person has started a new encounter (visit) to the Emergency Room.

- Single patient (identified by group membership) start of encounter

  This use case covers a system being notified when a patient within a defined group has started a new encounter.

  For example, a physician's system being notified that patients is being seen by another physician or organization.

## Canonical Argonaut `SubscriptionTopic` Resources

- SubscriptionTopic: [Encounter Start](canonical/subscriptiontopic-encounter-start.json)

- SubscriptionTopic: [Encounter End](canonical/subscriptiontopic-encounter-end.json)

## Define Profile on Subscriptions (channel, payload, etc)

- Subscription.channel.type = `rest-hook`

  Given the RESTful nature of FHIR, the primary communications channel chosen for this guide is also REST-based.  Although there are challenges in host REST services (e.g., forwarding notifications to phones), these challenges appeared to be manageable (e.g., a web server that converts REST notifications to Push Messages).

- Subscription.channel.payload = {`empty`, `id-only`}

  While the Subscription specification allows for three levels of payload information (`empty`, `id-only`, and `full-resource`), many implementers have expressed concerns around the security of including resources.  In order to comply with this guide, a server SHALL support, at a minimum, both the `empty` and `id-only` levels of payload information.

- Subscription.channel.payload.contentType = {`application/fhir+json`, `application/json`}

  Officially, the MIME type for JSON representations of FHIR resources `application/fhir+json`.  There are some differences from standard JSON, which are documented [here](http://hl7.org/fhir/json.html).  With this in mind, the preferred MIME type for notification bundles is `application/fhir+json`.

  Note that HTTP servers do not accept HTTP POSTS of application/fhir+json by default, and there will be clients which cannot reconfigure their systems to do so. For that reason, compliant servers SHALL also honor application/json as a valid MIME type for Subscriptions, bearing in mind that data MUST comply with the corresponding MIME type rules.

- Subscription.channel.end

  This field is necessary to detect stale subscriptions for removal.  Servers are allowed to determine a reasonable maximum time span, but MUST allow at least thirty one (31) days in the future.  The actual value used should be reasonably tied to the expected duration of the subscription, however it is understood that long-running Subscriptions will need to update this value periodically.  For example, on a server allowing the minimum span of thirty one (31) days, a Subscription created on 15 October 2019 would be allowed have an `Subscription.end` set for no later than 15 November 2019.

  Using a time less than one month is permitted.  For clients that expect to be relatively short-lived, it is reasonable to set a time less than one month in the future (e.g., one week, etc.).

  Servers SHALL support updating the `Subscription.end` field of an active Subscription so that a Subscription may be extended.  Servers are allowed to determine their maximum future span of time allowed when updating, given that it is also at least thirty one (31) days.

  When an expired Subscription is detected, a server may choose to either remove the resource or update the `status` to `off`.

## Minimum Required Filter Support

- Patient exact match

  One of the primary use cases for this guide is for a single patient to discover their own events.  To facilitate this, the canonical `SubscriptionTopic` instances for encounters allow for filtering on an exact match on `Patient.id` based on `Encounter.subject`.

- Group / List membership

  The other use case influencing this guide is based on a patient being declared within a `Group` or `List` resource (e.g., a resource specifying current patients for a care team).  To facilitate this, the canonical `SubscriptionTopic` instances for encounters allow for filtering on a `Patient` (via `Encounter.subject`) where a patient is `in` a `Group` or `List`.

## Security

- When sending notifications, servers SHOULD check that a notification is still authorized prior to sending it (e.g., client and user).
- Servers are responsible for ensuring that PHI is transmitted securely (e.g., should refuse to transmit in cleartext, require TLS, etc.).
- Servers SHOULD set a subscription state to `disabled` if a security validation fails, with an appropriate error message for diagnosis.
- Servers may want to consider the use of authenticated delivery systems (e.g., asymmetric key signing) to allow clients to validate the message origin without the need of a client secret.


## More on Subscriptions R5

Subscriptions in FHIR R4 were designed to be very generic and generally unconcerned with the internal operations of the server.  While powerful in concept, this design led to many servers not implementing Subscriptions.  Even on servers where it was implemented, it was generally restricted to certain areas or concepts, which were not able to be documented with R4 resources.  In practice, this meant that implementing Subscriptions required working extensively with a specific server implementation to discover and coordinate what functionality was usable and in which formats queries needed to be defined.

It was decided that the design of Subscriptions in R4 was not able to move beyond the issues, and that a redesign was required.  In FHIR R5, Subscriptions have been broken into two resources, `SubscriptionTopic` and `Subscription`, which address the issues with Subscriptions in R4.  Even with the redesign, there is a potential issue with interoperability based on Topic definitions, which this guide aims to prevent.

With the new design, servers are able to support a reduced set of Subscription Topics, with a reduced set of clearly-defined filters to clients.  This allows clients to be developed generically while also lowering the burden on server developers.

Subscription Topics fall under two categories: conceptual and computable.  There is overlap between the two (e.g., a server can computably define a new Admission encounter), but the server is able to use whatever internal implementation makes sense to capture the concept being expressed.  For example, in the Encounter-Start Subscription Topic defined below, two possible computable definitions are provided (in FHIRPath and using Query), but existing facade servers may
choose to simply implement the Subscription Topic as part of an existing HL7 v2 ADT workflow.

Given the goals of the redesign, filtering is based on Search and Search Modifiers.  This allows for more code reuse (for clients and servers) and lowers the bar for implementation.
