
# TODO
* 80 30
* scale up  a bit right column in slide 5

---

# Deep Dive into Nameko
## Demonstration of Nameko tools to implement a third-party system sync

<!--

Quick tour of some of the available libraries

on third party sync use case (plat - 3thd)

It's good if you are familiar with writing Nameko services
How to use entrypoints and dependency providers

TIM: Some of the things we demonstrate only quickly and you can come back to the source code Tim is going to run on his computer, it's on github .)

-->

---

# Salesforce Sync

![](https://raw.githubusercontent.com/iky/nameko-demo/master/slides/images/Nameko-demo-sf-sync.jpg)

{.column}

Salesforce is a CRM system, for us a third party system we integrate with.


<!--

A couple of services

Synchronising up and down

Tim has this setup on his computer (except Salesforce, cloud)

OK: "Tim, show the services code and run it please"

TIM shows the services code

TIM to audience: "Can you see a problem here?"

TIM shows the problem

-->

---

# Problem - infinite cycle

Solution:
* break the cycle when syncing up to salesforce
* break the cycle when syncing down from salesforce

<!--
with bi-directional thing we introduced potential looping problem
How to fix it...
-->


# Problem - infinite cycle

Break the cycle when syncing up to Salesforce

```yaml {style="font-size:10;"}
# salesforce/config.yaml

SALESFORCE:
    USERNAME: ${SALESFORCE_USERNAME}
    PASSWORD: ${SALESFORCE_PASSWORD}
    SECURITY_TOKEN: ${SALESFORCE_SECURITY_TOKEN}
    SANDBOX: ${SALESFORCE_SANDBOX:true}
```

{.column}

```python {style="font-size:10;"}
# salesforce/service.py

@event_handler('contacts', 'contact_created')
def handle_platform_contact_created(self, payload):
    result = self.salesforce.Contact.create(  # !!
        {'LastName': payload['contact']['name']}
    )

@handle_sobject_notification(
    'Contact', exclude_current_user=True,  # !!
)
def handle_salesforce_contact_created(
    self, sobject_type, record_type, notification
):
    self.contacts_rpc.create_contact(
        {'name': notification['sobject']['Name']}
    )
```

<!--
this way cycle is solved

we connect to sf API via a user

we can say ignore changes done by this user, so we do not get notification about a contact we just created

see the ignore here

what about the other way round
-->

---

# Problem - infinite cycle

Break the cycle when syncing down from Salesforce

Use of Nameko Worker Context

```python {style="font-size:9;"}
# worker context dump
{
    "call_id_stack": [
        "salesforce.handle_salesforce_contact_created"
        "contacts.create_contact",
        "salesforce.handle_platform_contact_created"
    ],
    # ...
    "sourced_from_salesforce": True,
}
```

{.column}

```python {style="font-size:9;"}
@event_handler('contacts', 'contact_created')
def handle_platform_contact_created(self, payload):
    if self.source_tracker.is_sourced_from_salesforce():
        logger.info("Ignoring event sourced from salesforce")
        return
    self.salesforce.Contact.create(
        {'LastName': payload['contact']['name']}
    )

@handle_sobject_notification(
    'Contact', exclude_current_user=True,  # !!
)
def handle_salesforce_contact_created(
    self, sobject_type, record_type, notification
):
    with self.source_tracker.sourced_from_salesforce():
        contact = self.contacts_rpc.create_contact(
            {'name': notification['sobject']['Name']}
        )
```

<!--

 more interesting - we are going to use nameko context here

who knows what what nameko worker Context is?

It is also used to pass metadata from call to call

call id stack (you’ll see in tracer, we’ll talk about later)

You can add custom stuff to it

OK: Tim please show how we are using it for breaking the cycle this way

[CUT to TIM]

Tim: explains in code
Next: Tim shows this working

[CUT to OK]

Show the worker context

OK: with great power comes great responsibility ;) don't abuse this!

Straight to the next slide ...
-->

---

# Redundancy

You usually run number of instances of the same service.

<!--

Next problem wee wanna show, solving redundancy issue

Jakob talked about using Kubernetes, we run number of instances of each service

What do you think to happen when a number of service instances listen to same salesforce events

Go to next slide, wait for answer from the audience ...

-->

---

# Redundancy

You usually run number of instances of the same service.

![](https://raw.githubusercontent.com/iky/nameko-demo/master/slides/images/Nameko-demo-sf-sync-down-redundancy.jpg)

<!--
Two entrypoints fire

We create two contacts

Go to the next slide
-->

---

# Redundancy

You usually run number of instances of the same service.

![](https://raw.githubusercontent.com/iky/nameko-demo/master/slides/images/Nameko-demo-sf-sync-up-and-down-redundancy.jpg)

<!--
You would have the same issue coming from the contact service

Nameko has this covered by default in its PUB/SUB API

I'll shortly talk about a bit later

Focus on the flow from Salesforce

Next: Tims shows the issue AND the solution in code

-->

---

# Redundancy - coming from Salesforce

Use of `ddebounce.skip_duplicates`

```python {style="font-size:10;"}
def skip_duplicate_key(
    sobject_type, record_type, notification
):
    return 'salesforce:skip_duplicate:{}'.format(
        notification['event']['replayId']
    )
```

(Opensourced not much documented look for tests)

{.column}


```python {style="font-size:10;"}
from ddebounce import skip_duplicates
from nameko_redis import Redis

class Service:

    redis = Redis()

    @handle_sobject_notification(
        'Contact', exclude_current_user=True,
    )
    @skip_duplicates(
        operator.attrgetter('redis'),
        key=skip_duplicate_key
    )
    def handle_salesforce_contact_created(
        self, sobject_type, record_type, notification
    ):
        self.contacts_rpc.create_contact(
            {'name': notification['sobject']['Name']}
        )
```

<!--

ddebounce library

not going into details

it's not mutext implementation, is rather naive,

but good enough for most of our cases

operations in syncing, most of the time are idempotent

it's easier if you implement your sync idempotent

-->

---

# Redundancy - coming from platform

Use of `handler_type` of Nameko event handler

`SERVICE_POOL` (default) vs `BROADCAST`

<!--
got to next slide ...
-->

---

# Debounce

A problem - many of the same (or similar) events fired (rapidly) in short period of time.

Debounce for the debounced method execution time.

<!--

A related problem ... read it out

Where one would debounce, Tim?

TIM says examples

TIM shows the solution / debounce the debounce applied on the entrypoints

((

update vs create (simplified  on create)

real live sync, don;t use payload (could hold stale info) gets the latest from source of truth instead

we've got just the sync queue, which just takes the primary key (the only event payload) and it then figures it out if sshould create or uppdate

))

go to the next slide

-->


---

# Retry

Things can go regularly wrong (Salesforce not responding, network issues) things you can recover from

<!-- 

Obviously, things can go wrong,

regularly


we need a way to recover from it!

What should we do Tim?

next: Tim shows retry

-->

---

# Retry

* Things can go regularly wrong
* Other use cases such as waiting for related objects to make it to the system though their own sync in an async world

{.column}

```python {style="font-size:10;"}
@entrypoint_retry(
	retry_for=(SalesforceError, RequestException),
	limit=10,
	schedule=(10000, 30000, 60000, 120000, 180000),
)
@entrypoint_retry(
	retry_for=RelationNotFound,
	limit=12,
	schedule=(2000, 5000, 10000, 10000, 10000),
)
@task
def handle_sync_from_platform(self, payload):
    # ...
```

<!--

(Real AMQP retry - handles system restarts, does not consume resources (it's not blocking, worker stops and new one is fired in the desired delay))
(update scenario again)

describe both points (on the left hand side)

Real salesforce service example (on the right hand side)

RelationNotFound, is raised trying to fill up a FK atrribute of an object

Student  - Contact example

can't relay on ordering

!! If you should take one thing out if this talk, it's retry ;)

It's under Nameko umbrella, you can find it there

What's next?

got to the next slide

-->

---

# Tracer

* Entrypoint calls tracing
* Uses python built-in logging
* Easy to set up
* We use ELK for collecting and looking up the traces

<!--

(don't read the slide yet .)

Jakob mentioned that we have a pretty good overview of what is happening [in our microservice stack

One of the reasons is that we do trace our entrypoint calls

We trace requests and responses

Tim, would you lake to show how we do it?

Next: Tim shows a little demo about
* how it works
* how easy this is to plug in

(don't forget to mention the id call stack and response time)

back to me - reading out the slide

ELK (Jakob's screenshot)

Tracer is the second thing to take from this talk!!! it's very powerful

-->

---

# autocrud and slack commands

<!--

couple of other things we wanted to show you ...

Tim shows and talks about it

* we repeat ourselfs when doing the sync for example
* does events too
* list - can you filter?

OK: Tim, what about commanding this from slack?

-->

---

# Thanks{.big}

Tim Buckland & Ondrej Kohout{.small}

https://github.com/timbu/nameko-demo{.small}

---

# Appendix

List of tools used:
```{style="font-size:12;"}
https://github.com/onefinestay/nameko-sqlalchemy
https://github.com/Overseas-Student-Living/nameko-salesforce
https://github.com/timbu/nameko-demo/blob/master/salesforce/source_tracker.py
https://github.com/etataurov/nameko-redis
https://github.com/Overseas-Student-Living/ddebounce
https://github.com/timbu/nameko-demo/blob/master/salesforce/tasks.py
https://github.com/nameko/nameko-amqp-retry
https://github.com/Overseas-Student-Living/nameko-tracer
https://github.com/timbu/nameko-autocrud
https://github.com/Overseas-Student-Living/sqlalchemy-filters
https://github.com/iky/nameko-slack
```
