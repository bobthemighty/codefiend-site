+++
title = "Tackling the Delivery Service refactoring kata"
slug = "tackling-delivery-service-refactoring-kata"
date = "2022-02-03"
+++

I've recently started a new role as an engineering coach, and I've been working through some katas to remember how to write code.

I'm a huge fan of Emily Bache's work, and was particularly interested in the [Delivery Service refactoring kata](https://github.com/emilybache/DeliveryController-Refactoring-Kata) because it neatly captures some of the challenges of working with legacy codebases.

I'm going to walk through the process I followed when solving the kata, and I'll include links to each commit so you can see the full code as we go. The repo is [here](https://github.com/bobthemighty/DeliveryController-Refactoring-Kata/)

The challenge set by the kata is to refactor a controller class so that we can replace some emails with text messages. The controller code is as follows:

```python,linenos
class DeliveryController:

    def __init__(self, delivery_schedule : list):
        self.delivery_schedule = delivery_schedule
        self.email_gateway = EmailGateway()
        self.map_service = MapService()

    def update_delivery(self, delivery_event : DeliveryEvent):
        next_delivery = None
        for i, delivery in enumerate(self.delivery_schedule):
            if delivery_event.id == delivery.id:
                delivery.arrived = True
                time_difference = delivery_event.time_of_delivery - delivery.time_of_delivery
                if time_difference < datetime.timedelta(minutes=10):
                    delivery.on_time = True
                delivery.time_of_delivery = delivery_event.time_of_delivery
                message = f"""Regarding your delivery today at {delivery.time_of_delivery}. How likely would you be to recommend this delivery service to a friend? Click <a href="url">here</a>"""
                self.email_gateway.send(delivery.contact_email, "Your feedback is important to us", message)
                if len(self.delivery_schedule) > i+1:
                    next_delivery = self.delivery_schedule[i+1]
                if not delivery.on_time and len(self.delivery_schedule) > 1 and i > 0:
                    previous_delivery = self.delivery_schedule[i-1]
                    elapsed_time = delivery.time_of_delivery - previous_delivery.time_of_delivery
                    self.map_service.update_average_speed(previous_delivery.location, delivery.location, elapsed_time)
        if next_delivery:
            next_eta = self.map_service.calculate_eta(delivery_event.location, next_delivery.location)
            message = f"Your delivery to {next_delivery.location} is next, estimated time of arrival is in {next_eta} minutes. Be ready!"
            self.email_gateway.send(next_delivery.contact_email, "Your delivery will arrive soon", message)



```

Phew! There's a lot going on here. I don't have any real context for understanding this code, except that it sends emails as part of a delivery service, and we want it to send SMS. Generally when I'm faced with a refactoring problem, I scan through the code - without reading in depth - looking for obvious clusters of functionality. There's enough happening here that I'm going to miss some nuances by reading the code, so I want to start testing my assumptions as quickly as possible. In this case, there's a lot of mingling of responsibilities, but we can still pick out some key ideas. 

* We need to find a delivery matching the delivery event and mark it as "arrived" (lines 9-12)
* We want to send a feedback email for the arrived delivery (16-17)
* If there is a next delivery, we want to send that recipient a message to let them know the currently estimated delivery time. (24-27)
* There's some stuff I don't really understand about late deliveries and maps

Refactoring is the process of improving the design of existing code, _guided by tests_. If you don't have test coverage for your changes, you're not refactoring, you're just rewriting. Before we can start to clean up this code, we need to get it under test. Step one is to write tests that describe the current behaviour. Once we've got those, we can start to change the code, knowing that we haven't broken the functionality.

## Goals

- [ ] Get the existing code under test without changing behaviour
- [ ] Make it so that we can replace the emails with texts
- [ ] See what else we can do to clean up the mess

First test looks like this:

```python
def test_a_single_delivery_changing_location():

    now = datetime.datetime.now()

    delivery = Delivery(
        id= 1,
        contact_email="fred@codefiend.co.uk",
        location=location1,
        time_of_delivery=now,
        arrived=False,
        on_time=True,
    )

    update = DeliveryEvent(1, now, location2)
    controller = DeliveryController([delivery])

    controller.update_delivery(update)

    assert delivery.arrived is True
    assert delivery.on_time is True
    assert delivery.location == location2
```

In this test, I create a new delivery expected at `now()` and deliver it to a different location at `now()`. My assumption is that I can change the location of a delivery with a DeliveryEvent.

Running the test, I get a scary looking error, though.

```bash
address = ('localhost', 25), timeout = <object object at 0x7fad46940d70>, source_address = None
ConnectionRefusedError: [Errno 111] Connection refused
```

Our test is failing because it's trying to talk to port 25 on localhost, and there's no service running there. Looking through the `DeliveryController` again, we can see that it's instantiating an instance of `EmailGateway` in its constructor. Port 25 is SMTP, so let's take a look at that EmailGateway class.


```python,hl_lines=14-16,linenos,hide_lines=4-11
class EmailGateway:
    def send(self, address, subject, message):
        msg = MIMEText(message)

        # me == the sender's email address
        # you == the recipient's email address
        msg["Subject"] = subject
        from_address = "noreply@example.com"
        msg["From"] = from_address
        msg["To"] = address

        # Send the message via our own SMTP server, but don't include the
        # envelope header.
        s = smtplib.SMTP("localhost")
        s.sendmail(from_address, [address], msg.as_string())
        s.quit()

```

Now we see the problem. The controller is creating a new instance of EmailGateway in its constructor, and the EmailGateway requires a running SMTP server. In an extreme situation, we could consider running an SMTP server in a docker container and capturing the emails that way, but for this code, we can easily fake out the dependency.

First step is to create our test double:

```python
class FakeEmailGateway:
    def send(self, address, subject, message):
        pass
```

Nothing too mysterious here. Our FakeEmailGateway is a Stub: an object that looks like another object, but has no real behaviour. I don't want to think right now about what the `EmailGateway`_does_ - I just don't want it to explode when it gets called.

Generally I prefer to write my own test doubles like this rather than using a mocking library. The advantage is that there is less overhead for each test, and I can make my doubles arbitrarily complex if I need to do something fancy. The trade-off is that I need to write a few lines of code before I get started. I find that's usually worthwhile, and often helps me understand the role of each dependency better.

Now we need to use our test double. First refactoring! We need to introduce a parameter to our constructor. We'll make the argument optional, like so:

```python,linenos,hl_lines=5
class DeliveryController:
    def __init__(
        self, 
        delivery_schedule: List[Delivery], 
        gateway: Optional[EmailGateway] = None
    ):
        self.delivery_schedule = delivery_schedule
        self.email_gateway = gateway or EmailGateway()
        self.map_service = MapService()
```


 I've made the argument optional because it means that any existing production code will continue to work, and will continue to use the existing EmailGateway class. Now in my test, I can create a new controller, passing my fake email gateway.

```python,linenos,hl_lines=16,hide_lines=2-14
def test_a_single_delivery_changing_location():

    now = datetime.datetime.now()

    delivery = Delivery(
        id=1,
        contact_email="fred@codefiend.co.uk",
        location=location1,
        time_of_delivery=now,
        arrived=False,
        on_time=True,
    )

    update = DeliveryEvent(1, now, location2)
    gateway = FakeEmailGateway()
    controller = DeliveryController([delivery], gateway)

    controller.update_delivery(update)

    assert delivery.arrived is True
    assert delivery.on_time is True
    assert delivery.location == location2
```

Running pytest again, our test runs, but fails.

```bash
=============================================== FAILURES ================================================
_______________________________ test_a_single_delivery_changing_location ________________________________

    def test_a_single_delivery_changing_location():
    
        ...
        
        assert delivery.arrived is True
        assert delivery.on_time is True
>       assert delivery.location == location2
E       AssertionError: assert Location(lati...de=21.0122287) == Location(lati...de=16.9251681)
E         
E         Differing attributes:
E         ['latitude', 'longitude']
E         
E         Drill down into differing attribute latitude:
E           latitude: 52.2296756 != 52.406374
E         ...
E         
E         ...Full output truncated (3 lines hidden), use '-vv' to show

test_delivery_service.py:39: AssertionError
```

The output here is interesting. We did mark the delivery as arrived, and the delivery was marked as on_time, but we don't update the location. It seems like the location in the `DeliveryEvent` is just ignored. With that new piece of knowledge we can rename our test and delete that last assertion.

```python
def test_a_single_delivery_is_delivered():
    ...
```

And we're green! [Commit d8ac1](https://github.com/bobthemighty/DeliveryController-Refactoring-Kata/commit/d8ac1c3ab8ced026b46dff94539a53f7b722fd09)

```bash
bob@bobs-spangly-carbon ~/c/p/D/python (main)> pytest
========================================== test session starts ==========================================
platform linux -- Python 3.10.2, pytest-6.2.5, py-1.11.0, pluggy-1.0.0
rootdir: /home/bob/code/play/DeliveryController-Refactoring-Kata/python
plugins: cov-3.0.0
collected 2 items                                                                                       

test_delivery_service.py ..                                                                       [100%]
```

Our goal here is to get coverage of the existing behaviour. We've successfully run a test by passing a Stub implementation for EmailGateway. It's not a test that we can rely on, though, because it isn't testing how the EmailGateway is used. For that we're going to need to extend our test. First we'll modify the `FakeEmailGateway`.

```python,linenos
class FakeEmailGateway:
    def __init__(self):
        self.sent = []
        
    def send(self, address, subject, message):
        self.sent.append((address, subject, message))
```

Now our `FakeEmailGateway` is a Spy object. It looks like an EmailGateway but it lets us spy on what the code is doing. When we call `send`, instead of doing nothing, we'll append the address, subject, and message to an internal list.

We can use that list to add some assertions to our existing test.

```python,linenos
    assert len(gateway.sent) == 1

    [(address, subject, message)] = gateway.sent
    assert address == "fred@codefiend.co.uk"
    assert subject == f"Your feedback is important to us"
    assert message == f"Regarding your delivery today at {now}. How likely would you be to recommend this delivery service to a friend? Click <a href=\"url\">here</a>"

```

We quickly run the tests again, and we're still green. I'm not _happy_ with this test because it repeats all the strings from the production code, and that's a smell, but for now I just want to get this class under test.

Before we make deeper changes, we need to make sure we've covered all the functionality. To do that, I'm going to use [pytest-cov](https://pypi.org/project/pytest-cov/), a tool for reporting on test coverage in python codebases. I'm not generally a fan of test coverage tools: if we're practicing TDD, we'll end up with high coverage by default. In this case, thgouh, we're writing tests after the fact, and it'll help me to understand which parts of the code aren't yet being exercised.

```bash
bob@bobs-spangly-carbon ~/c/p/D/python (main)> pip install pytest-cov
...
Installing collected packages: pytest-cov
Successfully installed pytest-cov-3.0.0
```

With this tool installed, we can run our test suite again and see what's not yet being tested.

```bash,hide_lines=3-6 14-16
bob@bobs-spangly-carbon ~/c/p/D/python (main)> pytest --cov . --cov-report term-missing
========================================== test session starts ==========================================
platform linux -- Python 3.10.2, pytest-6.2.5, py-1.11.0, pluggy-1.0.0
rootdir: /home/bob/code/play/DeliveryController-Refactoring-Kata/python
plugins: cov-3.0.0
collected 2 items                                                                                       

test_delivery_service.py ..                                                                       [100%]

---------- coverage: platform linux, python 3.10.2-final-0 -----------
Name                       Stmts   Miss  Cover   Missing
--------------------------------------------------------
delivery_controller.py        44      7    84%   50, 52-56, 60-64
email_gateway.py              12      8    33%   7-20
map_service.py                31      5    84%   24-25, 43-45
test_delivery_service.py      21      0   100%
--------------------------------------------------------
TOTAL                        108     20    81%


=========================================== 2 passed in 0.05s ===========================================
bob@bobs-spangly-carbon ~/c/p/D/python (main)> 
```
This report tells me that we're missing test coverage for lines 50, 52-56, and 60-64 of `delivery_controller.py`. Let's take a closer look and see if we can figure out what they do.

```python,linenos,linenostart=34,hide_lines=1-15 26-50
    def update_delivery(self, delivery_event: DeliveryEvent):
        next_delivery = None
        for i, delivery in enumerate(self.delivery_schedule):
            if delivery_event.id == delivery.id:
                delivery.arrived = True
                time_difference = (
                    delivery_event.time_of_delivery - delivery.time_of_delivery
                )
                if time_difference < datetime.timedelta(minutes=10):
                    delivery.on_time = True
                delivery.time_of_delivery = delivery_event.time_of_delivery
                message = f"""Regarding your delivery today at {delivery.time_of_delivery}. How likely would you be to recommend this delivery service to a friend? Click <a href="url">here</a>"""
                self.email_gateway.send(
                    delivery.contact_email, "Your feedback is important to us", message
                )
                if len(self.delivery_schedule) > i + 1:
                    next_delivery = self.delivery_schedule[i + 1]
                if not delivery.on_time and len(self.delivery_schedule) > 1 and i > 0:
                    previous_delivery = self.delivery_schedule[i - 1]
                    elapsed_time = (
                        delivery.time_of_delivery - previous_delivery.time_of_delivery
                    )
                    self.map_service.update_average_speed(
                        previous_delivery.location, delivery.location, elapsed_time
                    )
        if next_delivery:
            next_eta = self.map_service.calculate_eta(
                delivery_event.location, next_delivery.location
            )
            message = f"Your delivery to {next_delivery.location} is next, estimated time of arrival is in {next_eta} minutes. Be ready!"
            self.email_gateway.send(
                next_delivery.contact_email, "Your delivery will arrive soon", message
            )
```

Lines 49 and 50 say "if there's at least one more delivery in the schedule, get the next delivery". Line 51 asks "is this delivery late, and was there another delivery before it?"

Assuming all those things are true, then we're going to work out how long this delivery took, by comparing the time of this delivery and the previous one. Then we're going to call `map_service.update_average_speed` with the locations of the two deliveries and the elapsed time.

What about the other lines?

```python,linenos,linenostart=34,hide_lines=1-25
    def update_delivery(self, delivery_event: DeliveryEvent):
        next_delivery = None
        for i, delivery in enumerate(self.delivery_schedule):
            if delivery_event.id == delivery.id:
                delivery.arrived = True
                time_difference = (
                    delivery_event.time_of_delivery - delivery.time_of_delivery
                )
                if time_difference < datetime.timedelta(minutes=10):
                    delivery.on_time = True
                delivery.time_of_delivery = delivery_event.time_of_delivery
                message = f"""Regarding your delivery today at {delivery.time_of_delivery}. How likely would you be to recommend this delivery service to a friend? Click <a href="url">here</a>"""
                self.email_gateway.send(
                    delivery.contact_email, "Your feedback is important to us", message
                )
                if len(self.delivery_schedule) > i + 1:
                    next_delivery = self.delivery_schedule[i + 1]
                if not delivery.on_time and len(self.delivery_schedule) > 1 and i > 0:
                    previous_delivery = self.delivery_schedule[i - 1]
                    elapsed_time = (
                        delivery.time_of_delivery - previous_delivery.time_of_delivery
                    )
                    self.map_service.update_average_speed(
                        previous_delivery.location, delivery.location, elapsed_time
                    )
        if next_delivery:
            next_eta = self.map_service.calculate_eta(
                delivery_event.location, next_delivery.location
            )
            message = f"Your delivery to {next_delivery.location} is next, estimated time of arrival is in {next_eta} minutes. Be ready!"
            self.email_gateway.send(
                next_delivery.contact_email, "Your delivery will arrive soon", message
            )
```

Okay, so here we're saying "if there's a next delivery, then send an email to the recipient with the updated ETA". We use the `map_service` again to calculate what that ETA is, so we can assume that the `update_average_speed` call from above will affect the travel time. 

Great! So we've now read and understood the whole class. How do we get these lines under test?

Our first snippet, from 49-58 will be exercised if we have a delivery schedule where:

* There is a late delivery (line 51)
* that has at least one delivery after it (line 49)
* that is not the first delivery (also 51)

Assuming all those things are true, we should update our average speed on the map service.

Let's take a look at the MapService!


```python
class MapService:
    def __init__(self, average_speed=50):
        "average speed in km/h"
        self.average_speed = average_speed

    def calculate_eta(self, location1: Location, location2: Location) -> float:
        "return the number of minutes it will take to travel between locations at average speed."
        distance = self.calculate_distance(location1, location2)
        return distance / self.average_speed * MINUTES_PER_HOUR

    def calculate_distance(self, location1: Location, location2: Location) -> float:
        lat1 = radians(location1.latitude)
        lon1 = radians(location1.longitude)
        lat2 = radians(location2.latitude)
        lon2 = radians(location2.longitude)
        dlon = lon2 - lon1
        dlat = lat2 - lat1
        a = sin(dlat / 2) ** 2 + cos(lat1) * cos(lat2) * sin(dlon / 2) ** 2
        c = 2 * atan2(sqrt(a), sqrt(1 - a))
        # km
        distance = R * c
        return distance

    def update_average_speed(
        self, location1: Location, location2: Location, elapsed_time: datetime.timedelta
    ):
        distance = self.calculate_distance(location1, location2)
        updated_speed = distance / (elapsed_time.seconds / SECONDS_PER_HOUR)
        self.average_speed = updated_speed
```

Hmmm... The `update_average_speed` method takes the locations of two deliveries, calculates the distance between them, and uses that to work out the average speed. The `calculate_distance` function uses some trigonometry to derive the distance between two points on the globe. That's pretty cool, and I'm sure Emily knows what she's doing, but it's too complex for my needs.

I need to write a test to make sure that we call `update_average_speed`. Ideally I'd write something like:

```python
def test_when_the_delivery_is_late():
    ...
    controller.update_delivery()
    
    assert map_service.average_speed == 10
```

That's a little tricky, though, because to figure out what the average_speed should be, I'll have to do a bunch of maths, and I'm - frankly - too lazy. Instead, we'll repeat our trick from earlier, and inject a Spy.

```python
class FakeMapService:
    def __init__(self, value):
        self.value = value
        self.updates = []

    def calculate_eta(self, location1, location2):
        return self.value

    def update_average_speed(self, location1, location2, time):
        self.updates.append((location1, location2, time))
```

We inject this into our Controller just like the EmailGateway, making it optional so we don't break existing code. That means we can write a new test.

```python,linenos
now = datetime.datetime.now()
one_hour = datetime.timedelta(hours = 1)

def test_when_a_delivery_affects_the_schedule():
    """
    In this scenario, we have three deliveries to three locations.
    The first is scheduled to happen now, the second in an hour, the third in
    two hours.
    When the second delivery is delivered an hour late, we should call the map
    service to recalculate our average speed.
    """

    deliveries = [
        a_delivery(1, time=now, location=location1),
        a_delivery(2, time=now + one_hour, location=location2),
        a_delivery(3, time=now + (one_hour * 2), location=location3),
    ]

    gateway = FakeEmailGateway()
    maps = FakeMapService(10)
    controller = DeliveryController(deliveries, gateway, maps)

    controller.update_delivery(DeliveryEvent(2, now + (one_hour * 2), location2))

    [(start, end, time)] = maps.updates

    assert start == location1
    assert end == location2
    assert time == (one_hour * 2)
```

This test is green, so we have a working Spy. We've added a new utility function here `a_delivery` that builds us a `Delivery`object, setting default values for any fields we didn't specify. I'll often use this pattern to quickly create test data with less boilerplate.

[Commit 509b5a ](https://github.com/bobthemighty/DeliveryController-Refactoring-Kata/commit/509b5a180b9adffb4cc3bb37335d33f5e6948ef9)

```python
def a_delivery(id, contact_email="fred@codefiend.co.uk", location=location1, time=None,arrived=False, on_time=False):
        return Delivery(
            id=id,
            contact_email="fred@codefiend.co.uk",
            location=location,
            time_of_delivery=time or datetime.datetime.now,
            arrived=False,
            on_time=False,
        )
```

We want to write one more set of assertions: the next delivery recipient should receive an email with an updated ETA. We're in a good spot to do that, because our FakeMapService doubles up as a stub. Our `calculate_eta` method just returns the value we provided to the constructor.

All we need to do is set that value to something useful (say, 4 hours) and check that we send the correct email.

```python,hide_lines=2-15 21-27,linenos
def test_when_a_delivery_affects_the_schedule():
    """
    In this scenario, we have three deliveries to three locations.
    The first is scheduled to happen now, the second in an hour, the third in
    two hours.
    When the second delivery is delivered an hour late, we should call the map
    service to recalculate our average speed.
    """

    deliveries = [
        a_delivery(1, time=now, location=location1),
        a_delivery(2, time=now + one_hour, location=location2),
        a_delivery(3, time=now + (one_hour * 2), location=location3),
    ]

    gateway = FakeEmailGateway()
    maps = FakeMapService(480)
    controller = DeliveryController(deliveries, gateway, maps)

    controller.update_delivery(DeliveryEvent(2, now + (one_hour * 2), location2))

    [(start, end, time)] = maps.updates

    assert start == location1
    assert end == location2
    assert time == (one_hour * 2)

    [_,(address, subject, message)] = gateway.sent

    assert subject == "Your delivery will arrive soon"
    assert message == f"Your delivery to {location3} is next, estimated time of arrival is in 120 minutes. Be ready!"
```

Phew! With that, pytest-cov tells me I've reached 100% code coverage in the controller class.

## Goals

- [X] Get the existing code under test without changing behaviour
- [ ] Make it so that we can replace the emails with texts
- [ ] See what else we can do to clean up the mess

We've managed to get all the existing behaviour under test. The next step is to prepare the controller so that we can swap the email gateway for an SMS client. 

The design challenge is that SMTP and SMS aren't very much alike. When sending an email, we need an address and a subject to go along with our message, but when sending an SMS, we just need the recipient phone number.

We'll need to create a new abstraction that could represent _either_ an email or text client. Thankfully, this is a straightforward refactoring.

```python
class Notifier:
    """
    Abstract class for Notifiers. 
    """

    def request_feedback(self, delivery: Delivery):
        raise NotImplemented()

    def send_eta_update(self, delivery: Delivery, new_eta: int):
        raise NotImplemented()

class SmtpNotifier:
    """
    Implements Notifier by using an EmailGateway
    """

    def __init__(self, gateway: EmailGateway):
        self.gateway = gateway

    def request_feedback(self, delivery: Delivery):
        message = f"""Regarding your delivery today at {delivery.time_of_delivery}. How likely would you be to recommend this delivery service to a friend? Click <a href="url">here</a>"""
        self.gateway.send(
            delivery.contact_email, "Your feedback is important to us", message
        )

    def send_eta_update(self, delivery: Delivery, new_eta: int):
        message = f"Your delivery to {delivery.location} is next, estimated time of arrival is in {new_eta} minutes. Be ready!"
        self.gateway.send(
            delivery.contact_email, "Your delivery will arrive soon", message
        )
```

Here we've created a new class `SmtpNotifier` that wraps our EmailGateway and exposes two methods: `request_feedback` to send the feedback email, and `send_eta_update` to alert the next recipient.

In our tests we create a new SmtpNotifier passing a FakeEmailGateway, but the rest of our test remains the same. We can still check that the correct email ends up in the `sent` list on our fake, which proves that our notifier works correctly. If we wanted to, we could now add `contact_phone` to our delivery type, and write an SMSNotifier that would reimplement `request_feedback` and `send_eta_update`.

```python,hide_lines=9-11,hl_lines=5
def test_a_single_delivery_is_delivered():

    delivery = a_delivery(1)
    gateway = FakeEmailGateway()
    controller = DeliveryController([delivery], SmtpNotifier(gateway))

    controller.update_delivery(DeliveryEvent(1, now, location2))

    assert delivery.arrived is True
    assert delivery.on_time is True

    assert len(gateway.sent) == 1

    [(address, subject, message)] = gateway.sent
    assert address == "fred@codefiend.co.uk"
    assert subject == f"Your feedback is important to us"
    assert (
        message
        == f'Regarding your delivery today at {now}. How likely would you be to recommend this delivery service to a friend? Click <a href="url">here</a>'
    )
```

## Goals

- [X] Get the existing code under test without changing behaviour
- [X] Make it so that we can replace the emails with texts
- [ ] See what else we can do to clean up the mess

[Commit aa0518](https://github.com/bobthemighty/DeliveryController-Refactoring-Kata/commit/aa0518a2f6fb931fc6f8698d20366cc956a740ff)

Okay, let's take a look at the controller again. We've removed a little bit of cruft now that the messages are out, but it's still a mess.

```python
class DeliveryController:
    def __init__(
        self,
        delivery_schedule: List[Delivery],
        notifier: Optional[Notifier] = None,
        maps: Optional[MapService] = None,
    ):
        self.delivery_schedule = delivery_schedule
        self.map_service = maps or MapService()
        self.notifier = notifier or SmtpNotifier(EmailGateway())

    def update_delivery(self, delivery_event: DeliveryEvent):
        next_delivery = None
        for i, delivery in enumerate(self.delivery_schedule):
            if delivery_event.id == delivery.id:
                delivery.arrived = True
                time_difference = (
                    delivery_event.time_of_delivery - delivery.time_of_delivery
                )
                if time_difference < datetime.timedelta(minutes=10):
                    delivery.on_time = True
                delivery.time_of_delivery = delivery_event.time_of_delivery
                self.notifier.request_feedback(delivery)

                if len(self.delivery_schedule) > i + 1:
                    next_delivery = self.delivery_schedule[i + 1]
                if not delivery.on_time and len(self.delivery_schedule) > 1 and i > 0:
                    previous_delivery = self.delivery_schedule[i - 1]
                    elapsed_time = (
                        delivery.time_of_delivery - previous_delivery.time_of_delivery
                    )
                    self.map_service.update_average_speed(
                        previous_delivery.location, delivery.location, elapsed_time
                    )
        if next_delivery:
            next_eta = self.map_service.calculate_eta(
                delivery_event.location, next_delivery.location
            )
            self.notifier.send_eta_update(next_delivery, next_eta)
```

The first thing that sticks out at me is the loop. The `for i, delivery in enumerate` makes me sad for reasons I can't quite articulate. There's a bunch of places in the code where we're looking for `i + 1` or `i - 1` and it's kinda of hard to understand. What could we do instead?

What we're trying to do here is take a list of deliveries and work out whether there are deliveries before or after the one we're updating. Why don't we make that operation explicit?

```python
@dataclass
class ScheduleItem:

    item: Delivery
    prev: Optional['ScheduleItem'] = None
    next: Optional['ScheduleItem'] = None

    def find(self, id):
        """
        Walk the list looking for a delivery
        that has the given id.
        """
        if self.item.id == id:
            return self
        if self.next is not None:
            return self.next.find(id)


def build_schedule(deliveries: List[Delivery]) -> ScheduleItem:
    """
    Build a linked list of scheduled items.
    Each item contains a delivery and has an optional
    prev/next item.
    """
    prev = head = ScheduleItem(deliveries[0])
    for delivery in deliveries[1::]:
        curr = ScheduleItem(delivery)
        prev.next = curr
        curr.prev = prev
        prev = curr
    return head
```

The `ScheduleItem` class is a simple implementation of a doubly-linked list. With this data structure we can easily check whether we have a next or previous delivery, without the tedious looping and indexing.

```python,linenos
class DeliveryController:
    def __init__(
        self,
        deliveries: List[Delivery],
        notifier: Optional[Notifier] = None,
        maps: Optional[MapService] = None,
    ):
        self.delivery_schedule = build_schedule(deliveries)
        self.map_service = maps or MapService()
        self.notifier = notifier or SmtpNotifier(EmailGateway())

    def update_delivery(self, delivery_event: DeliveryEvent):
        scheduled = self.delivery_schedule.find(delivery_event.id)
        delivery = scheduled.delivery
        delivery.arrived = True

        time_difference = (
            delivery_event.time_of_delivery - delivery.time_of_delivery
        )
        if time_difference < datetime.timedelta(minutes=10):
            delivery.on_time = True
        delivery.time_of_delivery = delivery_event.time_of_delivery

        self.notifier.request_feedback(delivery)

        if not delivery.on_time and scheduled.next:
            previous_delivery = scheduled.prev.delivery
            elapsed_time = (
                delivery.time_of_delivery - previous_delivery.time_of_delivery
            )
            self.map_service.update_average_speed(
                previous_delivery.location, delivery.location, elapsed_time
            )
        if scheduled.next:
            next_delivery = scheduled.next.delivery
            next_eta = self.map_service.calculate_eta(
                delivery_event.location, next_delivery.location
            )
            self.notifier.send_eta_update(next_delivery, next_eta)
```

[Commit 50279c](https://github.com/bobthemighty/DeliveryController-Refactoring-Kata/commit/50279cfc356f6bbef828167bb621f06dcbbd0b29)

That's a _little_ better, but there's still a lot of noise. The next thing that's bugging me here is lines 15-23. There's a bunch of logic here for updating the state of our delivery that should probably live in our domain model.

Let's create a new method on the `Delivery` object to record the arrival. Again, this refactoring is trivial - we're just copying the code out of the controller and pasting it into the Delivery object:

```python,linenos,hide_lines=3-8
@dataclass
class Delivery:
    id: str
    contact_email: str
    location: Location
    time_of_delivery: datetime.datetime
    arrived: bool
    on_time: bool

    def arrive(self, time):
        self.arrived = True

        time_difference = time - self.time_of_delivery
        if time_difference < datetime.timedelta(minutes=10):
            self.on_time = True
        self.time_of_delivery = time
```

That makes a bigger difference. Our controller is down to the following

```python,hl_lines=4,linenos
    def update_delivery(self, delivery_event: DeliveryEvent):
        scheduled = self.delivery_schedule.find(delivery_event.id)
        delivery = scheduled.delivery
        delivery.arrive(delivery_event.time_of_delivery)
        self.notifier.request_feedback(delivery)

        if not delivery.on_time and scheduled.next and scheduled.prev:
            previous_delivery = scheduled.prev.delivery
            elapsed_time = (
                delivery.time_of_delivery - previous_delivery.time_of_delivery
            )
            self.map_service.update_average_speed(
                previous_delivery.location, delivery.location, elapsed_time
            )
        if scheduled.next:
            next_delivery = scheduled.next.delivery
            next_eta = self.map_service.calculate_eta(
                delivery_event.location, next_delivery.location
            )
            self.notifier.send_eta_update(next_delivery,next_eta)
```

We can definitely do better, though. On line 14, we're taking two delivery times and calculating the difference between them so that we can update our average speed. This feels like domain logic, too. Let's push that to our ScheduleItem. We can also move the logic for calculating the next eta.

[Commit 18276](https://github.com/bobthemighty/DeliveryController-Refactoring-Kata/commit/18276793e1117461184ab20851cdee0477f250aa)

```python,linenos,hide_lines=2-15
class ScheduleItem:

    delivery: Delivery
    prev: Optional["ScheduleItem"] = None
    next: Optional["ScheduleItem"] = None

    def find(self, id):
        """
        Walk the list looking for a delivery
        that has the given id.
        """
        if self.delivery.id == id:
            return self
        if self.next is not None:
            return self.next.find(id)

    def record_speed(self, maps: MapService):
        if not self.prev:
            return

        delivery = self.delivery
        prev = self.prev.delivery

        elapsed = delivery.time_of_delivery - prev.time_of_delivery

        maps.update_average_speed(prev.location, delivery.location, elapsed)

    def eta(self, maps):
        if not self.next:
            return
        
        return maps.calculate_eta(self.delivery.location, self.prev.delivery.location)
        
```
Using both of our new methods, brings the final shape of DeliveryController to:

```python,linenos
class DeliveryController:
    def __init__(
        self,
        deliveries: List[Delivery],
        notifier: Optional[Notifier] = None,
        maps: Optional[MapService] = None,
    ):
        self.delivery_schedule = build_schedule(deliveries)
        self.map_service = maps or MapService()
        self.notifier = notifier or SmtpNotifier(EmailGateway())

    def update_delivery(self, delivery_event: DeliveryEvent):
        scheduled = self.delivery_schedule.find(delivery_event.id)
        scheduled.delivery.arrive(delivery_event.time_of_delivery)

        self.notifier.request_feedback(scheduled.delivery)

        scheduled.record_speed(self.map_service)

        if scheduled.next:
            self.notifier.send_eta_update(scheduled.next.delivery, scheduled.eta(self.map_service))
```

That's definitely an improvement in my book!

## Key Takeaways

This was a fun kata, and it's useful to flex our legacy code muscles. We were given some code that was sort of hard to understand, so we used tests to check our assumptions about it, and to record the things we discovered.

We used a code coverage tool to make sure that we covered all the branches before we made any changes to the code.

Once we had covered the code, we introduced a new abstraction so that we could replace the EmailGateway with an SMSNotifier.

Lastly, since the code was fully tested, we could make radical changes to the structure. The biggest changes to the code were made without any changes to the tests. We pulled all the domain logic out of our controller and pushed it into a new datastructure while remaining green. The result - I think - is much easier to read and maintain.
