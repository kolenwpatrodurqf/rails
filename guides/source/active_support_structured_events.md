**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON <https://guides.rubyonrails.org>.**

Active Support Structured Events
================================

Active Support provides a unified API for reporting *structured events* inside your application. Structured events capture named facts about what happened, when it happened, and relevant contextual data. They form the foundation for Rails’ observability story and can be exported to any external provider.

This guide explains the concepts, design, and usage patterns behind structured events, and how Rails integrates them with framework instrumentation.

After reading this guide, you will understand:

* What structured events are
* When to use them
* How Rails reports events
* How subscribers consume and export events
* How tags and context work
* How structured subscribers connect Notifications → Events
* Where to find the authoritative API definitions
* The structured events emitted by Rails (framework hooks)

For the complete Event Reporter API, see:
**`ActiveSupport::EventReporter`** — [https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html](https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html)

-------------------------------------------------------------

Introduction to Structured Events
---------------------------------

Structured events are machine-friendly, semantically consistent records describing something that occurred in your application. They are designed for production observability, fast search, analytics platforms, and developer-facing diagnostic tools.

Each event contains:

* a **name**
* a **payload** (hash or event object)
* **timestamp** (nanosecond precision)
* **source location**
* **tags**
* **context**

Rails automatically attaches timestamps, source location, and context so developers only need to provide a name and payload.

### Why structured events?

Rails already emits unstructured logs. They are ideal for development but become difficult and expensive to use in production:

* logs from many threads and processes interleave
* no consistent schema
* specialized knowledge needed to query
* indexing costs grow with volume
* onboarding requires understanding bespoke log formats

Structured events fix these problems:

* consistent shape
* semantic naming
* predictable payload keys
* cheap to index in structured stores
* easy to query
* compatible with any O11Y backend

They also unify **business events** and **developer observability events** behind a single API.

---

Reporting Events
----------------

The canonical entry point is:

```ruby
Rails.event.notify("event.name", { id: 123 })
```

This attaches metadata and forwards the event to any configured subscribers.

For full method signatures and advanced options (debug, caller depth, filtering rules), see the EventReporter API documentation.

### Event names

Use namespaced identifiers:

```
"controller.request_started"
"auth.user_created"
"graphql.query_resolved"
```

Names should reflect *what happened*, not how or why.

### Event payloads

Payloads may be:

* simple Ruby hashes
* domain-specific event objects that define their own schema

Rails merges any additional keyword arguments into the payload hash.

For details on object payloads and automatic key normalization, see:
**Event Objects** — [https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Event+Objects](https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Event+Objects)

### Debug events

Debug events are emitted only when debug mode is enabled:

```ruby
Rails.event.debug("cache.evicted", size: 1024)
```

Use these for high-volume diagnostic telemetry.

For details:
**Debug Events** — [https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Debug+Events](https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Debug+Events)

---

Subscribers
-----------

A subscriber receives events and is responsible for exporting them to a destination such as:

* a log file
* a streaming platform
* a metrics or analytics system
* a SaaS observability backend

Example:

```ruby
class JSONSubscriber
  def emit(event)
    LogExporter.write(JSON.generate(event))
  end
end

Rails.event.subscribe(JSONSubscriber.new)
```

Subscribers may also use filter procs to receive only certain events.

See:
**Subscribers** — [https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Subscribers](https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Subscribers)
**Filtered Subscriptions** — [https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Filtered+Subscriptions](https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Filtered+Subscriptions)

---

Tags
----

Tags annotate an event with domain-specific labels:

```ruby
Rails.event.tagged("graphql") do
  Rails.event.notify("user.created", id: 123)
end
```

Resulting event:

```ruby
tags: { graphql: true }
```

Tags are stack-based and temporary.

See:
**Tags** — [https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Tags](https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Tags)

---

Context
-------

Context expresses execution-scoped metadata, such as request identifiers or job metadata. Context is automatically reset between requests and jobs.

```ruby
Rails.event.set_context(request_id: request.uuid)

Rails.event.notify("checkout.succeeded", id: order.id)
```

Resulting event:

```ruby
context: { request_id: "abcd123" }
```

For custom stores and behavior:
**Context Store** — [https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Context+Store](https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Context+Store)

---

Relationship to ActiveSupport::Notifications
--------------------------------------------

Although both systems relate to what happens inside the application, they solve different problems.

### ActiveSupport::Notifications

* an *instrumentation* API
* measures *duration*, *allocations*, *start/finish*
* publishers wrap blocks of code
* subscribers react synchronously

### Structured Events (Event Reporter)

* a *reporting* API
* describes *what happened*, semantically
* payloads are structured
* metadata is attached automatically
* built for observability systems
* subscribers export data externally
* designed for high cardinality, low-cost search

### How they work together

`ActiveSupport::StructuredEventSubscriber` bridges the two systems:

1. listens to Notifications (e.g., `"process_action.action_controller"`)
2. transforms them
3. emits structured events via `Rails.event.notify`

Framework components such as Action Controller, Active Record, Active Job, and others use structured subscribers to convert internal instrumentation into structured events.

---

Structured Event Subscribers
----------------------------

A structured subscriber is a specialized Notifications subscriber that turns instrumentation events into structured events.

Example:

```ruby
class StructuredControllerSubscriber < ActiveSupport::StructuredEventSubscriber
  attach_to :action_controller

  def start_processing(event)
    emit_event("controller.request_started",
      controller: event.payload[:controller],
      action: event.payload[:action],
      format: event.payload[:format]
    )
  end
end
```

Key behaviors:

* auto-names methods based on the notification name
* supports silencing “debug-only” events
* forwards errors to `ActiveSupport.error_reporter`

These subscribers form the backbone of Rails’ built-in events.

---

Security
--------

Hash-based payloads are filtered automatically using `config.filter_parameters`.

Event objects must be filtered by the subscriber.

See:
**Security** — [https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Security](https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html#label-Security)

---

Framework Hooks (Structured Events Emitted by Rails)
----------------------------------------------------

Rails emits structured events across the framework covering controllers, jobs, database activity, caching, mailing, streaming, and more.

A full catalog will appear in this section, mirroring the style used in the ActiveSupport::Notifications guide:

* event name
* payload keys
* example payload
* when the event occurs

TODO: Fill in this section.

