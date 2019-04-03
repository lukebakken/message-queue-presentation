<!-- vim:sw=4:tw=144:expandtab:
-->

## Initial use case

We need fizzbuzz.com to allow creation of user accounts. People sign up and provide the following information:

* Name
* Address
* Phone Number
* Email

Pretty standard. Here's what a web controller might look like:

```
HtmlResult create_user(HtmlFormData fd, IDbRepository db) {
    var db_usr = new DatabaseUser(fd.Name, fd.Address, fd.Phone, fd.Email);
    db.Create(usr);
    return new CreateUserSuccessView(usr);
}
```

Things to keep in mind:

* Super happy path. No exception handling.
* Database could be unavailable.
* Everyone could sign up at once. What happens then?
* What else?

## FizzBuzz 2.0!

Marketing comes back and is very excited about the sign up experience at FizzBuzz. But, they'd love it if the new user were sent an email with
the current FizzBuzz newsletter, and maybe a text message greeting too. Users love that.

So, FizzBuzz dev updates the code for creating a new user. Now, all those services would be injected in the ctor most likely but you get the
idea since this is coded in PseudoSharp™:

```
HtmlResult create_user(HtmlFormData fd, IDbRepository db, ITwilioManager tmgr,
                       IEmailService esvc, IObjectBlobService blobs) {
    var usr = new UserModel(fd.Name, fd.Address, fd.Phone, fd.Email);
    try {
        var db_usr = new DatabaseUser(usr);
        db.Create(usr);

        var text_data = new TextMessage(usr, "Welcome to FizzBuzz!");
        tmgr.Send(text_data);

        var newsletter = blobs.LoadCurrentNewsletter();
        var email_data = new EmailMessage(usr, "Welcome to FizzBuzz!", newsletter);
        esvc.Send(email_data);

        return new CreateUserSuccessView(usr);
    } catch (FizzBuzzException ex) {
        return new CreateUserErrorView(usr);
    }
}
```

Much more going on in version 2.0! What can we say about the additions?

* Several services, any of which could be unavailable.
* What happens if user is saved in database but the next step fails?
* Everyone could still sign up at once! Still could overload everything.
* What else?

## Can Haz Q

After rolling out FizzBuzz 2.0 the email server went down several times and new users didn't get their copy of the newsletter. Support tickets
went through the roof and finally the CTO went to the dev team and said "Fix it!". So the team read about message brokers and installed one in
their environment. Now the code looks like this:

```
HtmlResult create_user(HtmlFormData fd, IMessageQueue mq) {
    var usr = new UserModel(fd.Name, fd.Address, fd.Phone, fd.Email);
    var event = new CreateUserEvent(usr);
    mq.PublishEvent(event);
    return new UserCreationInProgressView(usr);
}
```

That's pretty nice, but what happened to the other actions? Several new services were deployed (Unix daemons / Windows services) that subscribe
to the `CreateUserEvent` event. Here's the gist of it for the database service and the email service:

* Database service:

    ```
    void ServiceStart(IMessageQueue mq) {
        mq.Subscribe("user.create");
    }

    void HandleEvent(IMessageQueueEvent evt, IDbRepository db) {
        var db_usr = new DatabaseUser(evt.Data);
        db.Create(db_usr);
    }
    ```

* Email service:

    ```
    void ServiceStart(IMessageQueue mq) {
        mq.Subscribe("user.create");
    }

    void HandleEvent(IMessageQueueEvent evt, IEmailService esvc, IObjectBlobService blobs) {
        var usr = new UserModel(evt.Data);
        var newsletter = blobs.LoadCurrentNewsletter();
        var email_data = new EmailMessage(usr, "Welcome to FizzBuzz!", newsletter);
        esvc.Send(email_data);
    }
    ```

It's safe to say that we've gained a lot here:

* Create user sub-operations proceed in parallel, which is great, but there is still an issue with that... what is it?
* System _should be_ able to handle greater load (the "everyone signs up at the same time scenario"). Why is that?
* Adding new features should be easier. We could tweet every new user signup! Users love that.

## Wait a second, why a message broker and not ...

Instead of publishing an event, calls to other services (via HTTP, TCP sockets, etc) could have been made (in parallel, even). What are some of the advantages of using
a message broker instead of parallel service requests?

* If the destination service is down in the above scenario, the publisher must take that into account by saving the data and re-trying in the future. With a
message broker, once the data is published and confirmed, you don't have to worry about what happens after. The data will remain in the
designated queue(s) until consumed (and acknowledged). The more services you must call, the more you have to keep track. Publishing a single
message is much easier.
* Message brokers are designed to keep statistics about message rates. You can learn a lot about your overall system by monitoring this data.
Logs can be directed through the broker, as well as exceptions. You can get stats about a lot of things for "free".
* Easy to add multiple instances of queue consumers and have the broker round-robin message delivery to each for scaling out.
* Need to add a feature without changing data? Just consume from a queue as we have seen above.

## Considerations

### Order of operations

We probably want to ensure the user has been created in the database prior to doing other operations. How could this be achieved? Here's one
way:

* Only the database service listens for the `user.create` event.
* When the DB finishes successfully, `user.created` is published to the broker.
* Other services listen for a subsequent event, like `user.created`.

Example database service code:

```
void ServiceStart(IMessageQueue mq) {
    mq.Subscribe("user.create");
}

void HandleEvent(IMessageQueueEvent evt, IMessageQueue mq, IDbRepository db) {
    var db_usr = new DatabaseUser(evt.Data);
    db.Create(db_usr);
    var event = new UserCreatedEvent(evt.Data);
    mq.PublishEvent(event);
}
```

### Async UI

Now that the process of creating a user happens asynchronously, how can the UI be notified when their account is ready? Here are some options I
can think of -

* Don't notify them via the web UI! This is the simplest option. Return a page stating that their account will be created and they will receive a
text message and email with information about their new account.
* But, let's say marketing wants to notify the user on the web page if they remain on it during the account creation. Now that web sockets
exist, this is somewhat trivial to implement. The `UserCreationInProgressView` page opens a web socket back to the web server and listens for the
`user.created` event. The RabbitMQ Web STOMP and web MQTT plugins are designed for this use-case.

# Links

https://www.cloudamqp.com/blog/2014-12-03-what-is-message-queuing.html

https://stackify.com/message-queues-12-reasons/

https://www.cloudamqp.com/blog/2018-12-10-what-its-like-to-bet-your-entire-startup-on-rabbit.html

https://ayende.com/blog/186849-A/production-ready-code-is-much-more-than-error-handling
