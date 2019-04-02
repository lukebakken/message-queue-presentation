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
    var usr = new UserModel(fd.Name, fd.Address, fd.Phone, fd.Email);
    db.Create(usr);
    return new CreateUserSuccessView(usr);
}
```

Things to keep in mind:

* Super happy path. No exception handling.
* Database could be unavailable.
* Everyone could sign up at once.
* What else?

## Fizzbuzz 2.0!

Marketing comes back and is very excited about the sign up experience at FizzBuzz. But, they'd love it if the new user were sent an email with
the current FizzBuzz newsletter, and maybe a text message greeting too. Users love that.

So, FizzBuzz dev updates the code for creating a new user:

```
HtmlResult create_user(HtmlFormData fd, IDbRepository db, ITwilioManager tmgr, IEmailService esvc, IObjectBlobService blobs) {
    var usr = new UserModel(fd.Name, fd.Address, fd.Phone, fd.Email);
    try {
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
* Everyone could still sign up at once!
* What else?


# Links

https://www.cloudamqp.com/blog/2014-12-03-what-is-message-queuing.html

https://www.slideshare.net/old_sound/pivotal-labs

https://stackify.com/message-queues-12-reasons/

https://www.cloudamqp.com/blog/2018-12-10-what-its-like-to-bet-your-entire-startup-on-rabbit.html
