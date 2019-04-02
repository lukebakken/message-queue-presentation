<!-- vim:sw=144:tw=144:expandtab:
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
    var db_model = new UserModel(fd.Name, fd.Address, fd.Phone, fd.Email);
    db.Create(db_model);
    return new CreateUserSuccessView(db_model);
}
```

Things to keep in mind:

* Super happy path. No exception handling.
* Database could be unavailable.
* Everyone could sign up at once.
* What else?

## Fizzbuzz 2.0!

Marketing comes back and is very excited about the sign up experience at FizzBuzz. But, they'd love it if the new user were sent an email with
the current FizzBuzz 


# Links

https://www.cloudamqp.com/blog/2014-12-03-what-is-message-queuing.html

https://www.slideshare.net/old_sound/pivotal-labs

https://stackify.com/message-queues-12-reasons/

https://www.cloudamqp.com/blog/2018-12-10-what-its-like-to-bet-your-entire-startup-on-rabbit.html
