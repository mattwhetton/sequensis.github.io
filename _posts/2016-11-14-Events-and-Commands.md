---
layout: post
title:  "Events and Commands"
date:   2016-11-14 13:00:00 +0100
categories: NServiceBus
---

# Commands and Events

## Purpose

To provide clear guidelines on the usage of events and commands, and the Sequensis approach to this. 

*Note that this is a living document and should continue to evolve and change in line with improvements to bets practice within Sequensis*

### This article will:
- Help new starters get up to speed with events and commands at Sequensis
- Clarify what we mean by events and commands
- Provide a point of reference for PR's
- Provide a point of reference for new service implementations

### This article will not
- Address message versioning. This will be addressed in a separate document.
- Provide low-level technical implementation details. This is covered by the [NServiceBus Website] and book
- *Cover the topic of context interchange and domains that span multiple components, although it may be updated in the future to do so.*

## Definition of events and commands
Refer to the [NServiceBus Website on events and commands] for a more detailed guide on what events and commands are, and how to implement them. Some key features are that:

 - **Events**: Published in one place, handled in many; Can be subscribed to in a pub/sub pattern.
 - **Commands**: Sent from many places, handled in only one place;  Cannot be subscribed to.

## Guidelines

### 1. Preference for Events
In general we should prefer using events over commands, although there are specific use cases where commands should be used. The reasons for this are as follows:

#### 1.1 **Flexibility**: 
Events give more flexibility and extensibility, as they can be subscribed to make other services in future use cases. Commands are much stricter and only satisfy a single use case.

#### 1.2 **Domain Centric Language**:  
Events encourage a richer, more domain driven language, describing things that have happened / are happening. Commands tend to be more asking specific components to do specific things.

#### 1.3 **Coupling**: 
Whilst using a service bus does technically decouple our implementations,  commands tend to drive a more tightly coupled architecture, as service depend on one another more. Events encourage more loosely coupled architecture as components have less knowledge of one another. 

### 2. Event Guidelines
The following is some high level guidelines on the usage of events:

#### 2.1. **Domain Naming Representation**
Thought needs to be given to the naming of events, to clearly explain what has happened in the domain. For example 
- It is ok to raise an event called UnsubscribeApplicantRequested before the action has taken place in the database as it represents that the applicant has indicated they wish to unsubscribe. This is an event representing something that has happened in the domain, even if the wider system has not yet processed it.
- It is not ok to raise an event saying EmailSent before an email has actually been sent, as this is not representing what has actually happened in the domain.
	
#### 2.2 **Context Boundary**
The namespace portion of the event is actually part of the event name that implies context. For example:
- A TopUpLoanService.ApplicationCreated event is when an application is created in the context of the top-up loan service. In this case the event is ApplicationCreated and the context boundary is the TopUpLoanService.
	
#### 2.3 **Naming Specificity**
A preference for specific messages vs general message with discriminator, for example:
- A general message with discriminator might be ApplicationStatusChanged with a property of Reason (Declined, Approved etc).
- Specific messages would be ApplicationDeclined, ApplicationApproved etc

**Reasoning**
- General messages introduce unnecessary branching logic in handlers, which would be better addressed by handling a specific event
- General messages introduce more traffic, as interested components are required to subscribe to the higher level event (and therefore all instances that are raised), whereas specific messages allow interested components to only subscribe in the specific messages they need.

#### 2.4 **Event Ownership**
- Event ownership is denoted by the namespace they exist within, for example the TopUpLoanService.ApplicationCreated event is owned by the TopUpLoanService. 
- Events can only be raised by the owner of that event. In the above example only the TopUpLoanService can raise the TopUpLoanService.ApplicationCreated event.

### 3. Command Guidelines
Commands are generally used to introduce stability and leverage standard errors handling / retries. Therefore given that commands are limited in flexibility they should always:

#### 3.1 **Atomic Operations**
Be atomic and transactional (i.e. used to wrap low level operations)

#### 3.2 **Single Action**
Wrap a single action. (where action represents something that changes a resource or impacts pricing, whether external or internal)

The following is some guidelines to when it might be appropriate to use commands.

#### 3.3 **Multi-Step Process**
The use of internal commands to break down multi-step in event handlers is ok. For example
- The HighLevelChecksPassedEventHandler might call the Experian bureau service, but persist the data using an internal StoreExperianBureauResultCommand. This isolates the performance of the request, from any failures in the saving of the data.

#### 3.4 **Isolate Resource**: 
The use of commands to isolate an specific resource is also ok, for example:
- A command such as SendEmail to isolate the email sending functionality.

#### 3.5 **Co-ordinating Event Handler**:  
The use of internal commands with a co-ordinating event handler as opposed to multiple event handlers is preferred. In this case there would be a single event handler that would raise several internal commands, and therefore just act as a co-ordinator. For example:
- The ExperianBureauService handles the HighLevelChecksPassed event and raises commands to perform the ExperianBureauSearch and ExperianAuthenticateSearch.

**Reasoning**
- The main risk in having several event handlers in the same host is that if one event handler fails, all are retried. This increases the risk of double handling. Whilst there is a risk of a co-ordinating handler failing part way through, it is less likely than event handlers that execute actual business logic.
- Regardless of the approach, idemptoency needs to be considered and accounted for accordingly.

### 4. Other Considerations

#### 4.1 **Idempotency**
All handlers should be idempotent, whether through the NServiceBus outbox, the idempotency manager or other means.

#### 4.2 **Retry policies**
Consideration should be given to retry policies and what exceptions are thrown within handlers. For example, a timeout exception might be ok as a retry might fix the issue, whereas an argument null exception probably isn't as retrying wouldn't fix the null argument.

**TBD: Follow up with guidelines around how to throw an exception that forces the event to the error queue**

#### 4.3 **Poison message handling**
Thought needs to be given to poison messages when handling validation and throwing exceptions. i.e. throwing exceptions that retries cannot fix should be avoided. For a full definition of poison messages see [poison message definition]

#### 4.4 **Raise an event from a handler**
Raising a data rich event at the end of a handler to broadcast what has been done is generally good practice, but not a requirement.

#### 4.5 **Raising multiple events from a handler**
Raising multiple events from a single handler is generally considered a code smell, as it would indicate that the handler is completing more than one action.

## Use your judgement
The final thing to note is that these are guidelines and should not be followed dogmatically. Use your common sense and depart from the guidelines if necessary - as with all guidelines, should you do so it would make sense to make note of the departure and why in the project (either near the code or in the readme). Getting opinions from around the team is also highly recommended as it brings different points of view, gains consensus across the team and may identify further best practice.

[NServiceBus Website]:https://docs.particular.net/nservicebus/
[NServiceBus Website on events and commands]: https://docs.particular.net/nservicebus/messaging/messages-events-commands
[poison message definition]: https://msdn.microsoft.com/en-us/library/ms789028(v=vs.110).aspx






