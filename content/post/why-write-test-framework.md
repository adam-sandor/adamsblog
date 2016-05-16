+++
date = "2016-05-12T12:42:04+02:00"
draft = true
title = "Why you should build your own test DSL"
+++

I'll start with this: test automation is cool. Fortunately these days nobody denies that. We need nearly all our testing to be automated to support the short iterations of agile development. What I still see often however is test code that isn't given the same care as production code because.. well.. it's just test code right? It's true that badly written test code won't cause a production outage but if you are serious about your testing and write a lot of tests the test code can become a huge maintenance burden. I'm going to focus on the upper part of the testing pyramid - tests that are interchangeably called acceptance or functional tests but the same principles can be applied to other types of tests too.

A typical application would be tested using it's UI and some exposed webservice APIs. The straightforward approach to testing is to use the test frameworks available to us and write some tests using them. Here I'll be using [Groovy HttpBuilder] to submit an order to a webshop's API webservice and [Geb](http://www.gebish.org) to verify if the order has appeared in the admin UI. I don't try to make this actually working code but I also didn't want to go pseudocode all the way.

First to give you an idea of what the application and the interaction we are describing looks like:
![Application layout](/images/applayout.png)

Simulate remote request for new purchase:
``` groovy
def client = new groovyx.net.http.RESTClient('http://myshop.com/api')
def resp = client.post( path : '/order/new',
                        body : [productCode: 'ABCD123', amount: 10] )

assert resp.status == 200
def newOrderLocation = resp.headers['Location']
assert newOrderLocation != null
```

Verify if new order appears on the Admin UI:
``` groovy
geb.Browser.drive {
  go "http://myshop.com/admin/login"
  $("form.login").with {
      $("#username") = "admin"
      $("#password") = "password"
      $("#login").click()
  }
  assert $("h1").text() == "Admin Section"

  go newOrderLocation //the location from the rest request above
  assert $('#product-code').text() == "ABCD123"
  assert $('#amount').text() == "10"
}
```

This pretty neat code using frameworks with nice APIs. The problem is not the quality of the code but the level of abstraction. Looking more closely at this code it turns out it's making lots of assumptions:

- A new order is a post request on the url /order/new
- The new order's ID is returned in the Location header
- The admin page login is on /admin/login
- IDs of the elements on the login and admin page.

Imagine you'll have to write tens or even a hundred of such tests, most of them more complex than my simple example. These assumptions will be repeated over and over in every test and when changes to the application break one of them you'll have a lot of refactoring to do. Those of you who have used Geb before are probably screaming [Page Object pattern](http://www.gebish.org/manual/current/#the-page-object-pattern) by now and you are right. UI test frameworks (at least the ones I know) have come up with a way to abstract element identification from tests by defining a set of Page Objects. Our test would have a LoginPage and an AdminPage which would encapsulate the underlying elements. Using Page Objects our test would look like this:
``` groovy
geb.Browser.drive {
  to LoginPage
  username = "admin"
  password = "password"
  login()
  go newOrderLocation
  productCode == "ABCD123"
  amount == 10
}
```
This is a big improvement - all element queries are gone, the DOM model of the page nicely encapsulated. This level of abstraction will suffice for many applications, but I would like to focus on those for which it doesn't.

### One step further - Application driver pattern

The previous way of defining a test can be improved by raising the level of abstraction even more to deal with some remaining issues:

1. All the details of our webservice API are visible in the test. This being a public API this might not seem like a big issue but development will have a stage when there will be frequent changes even to public APIs. Also distributed applications might require accessing of internal APIs in tests too.
- The UI part of the test defines a user journey but the only thing we are interested in is whether the order appeared on the page not how the user got there. (Of course there will be tests where the user journey is the target of the test)

Let's see how a test could look like if we removed those last unnecessary details:
``` groovy
def app = new myshop.test.ApplicationDriver(http://myshop.com, "username", "password")
myshop.test.Order order = app.api.newOrder(productCode: 'ABCD123', amount: 10)
app.ui.orders.verifyIfPresent(order)
```

Here we have built a complete DSL (both are valid ways to look at it) around our application. A tremendous amount of detail can be hidden in these two calls but thats what we want to achieve. Now if the URL structure of the API webservice or the way to log in to the application changes our test will stay intact, only the framework needs to change. This is described in Jez Humble and Dave Farley's book [Continuous Delivery](http://www.amazon.com/Continuous-Delivery-Deployment-Automation-Addison-Wesley/dp/0321601912 as the `Application Driver pattern`.

What's also interesting is that the `ApplicationDriver` takes care of state handling. It's the central component that know the basics about the application like the URL and login credentials so sub-components like the `UIDriver` and the `ApiDriver` don't have to bother with those.

You can see that I explicitly specified the packages for the ApplicationDriver and the Order classes. This is to show a lesson I learned during implementing such test frameworks/DSLs. You shouldn't reuse classes from the production code base. It not surprising that `ApplicationDriver` is in the `myshop.test` package but there probably already is a class for representing `Orders`. My advice: don't reuse it. Coupling the production code to the testing code will create unnecessary problems down the road when evolving one or the other will be hard. Some code duplication can be a good thing if it means independent components can be evolved really independently, and the test framework/DSL is definitely such a component.

### In summary - The stack

To sum it up here is a diagram that show how the different layers build up the tests. There is a bit of cheating here as the support frameworks are not built on top of the test runners but it just looks nicer like this :)

![Test application stack](/images/teststack.png)

### Made for testers

Another important aspect is the simplification of the test itself. While for the previous implementation needed knowledge of two frameworks the new only requires knowledge of the application specific DSL. This DSL will by nature be quite limited so learning it should be straightforward. This means testers with less programming knowledge will be able to write tests using the framework. This can be very useful for a lot of organizations. Having access to such a tool that is programming but in a simplified way can be a nice stepping stone for testers coming from a non-technical background but interested in test automation.

### The price to pay - flexibility

Flexibility is the price that needs to be payed for such a framework. Test writers (be it developers, BAs or testers) can only use the available methods of the DSL. It's the least problem for developers who can add new stuff when they need it but testers and BAs will have it harder. There needs to be a good cooperation between the developers and the testers/BAs to define and quickly implement new features. Furthermore the design of the DSL is no small feat and in the initial phase when lots of features are being added it can be challenging to keep everything tidy and well structured.

My advice is to not start building the DSL right at the start but rather as with building any complex framework wait until you properly understand the problem and see the right abstractions. On the other hand don't wait until you have a hundred tests ;)

### BDD Frameworks

So how does this relate to BDD frameworks like [Cucumber](https://cucumber.io)? I'd say from a birds eye view they are the same. Both BDD frameworks and Application driver provide a high level abstraction over the application so tests can be detached from application details. But of course the devil is in the details... The Application driver approach is a programmers approach for tackling changing application internals and producing a nice DSL that can be used by non-developers as a useful side-effect. The BDD approach primarily wants to foster test definition by non-developers. I must admit that I never tried any BDD framework much less the full blown approach with trying to make BAs write non-code looking tests. I can see and even larger investment necessary + a steep learning curve with uncertain results. By uncertain results I mean whatever pseudo-code you'll give the BAs to use will still have it's tight rules to make it executable by a machine. If you have lazy BAs you might end up writing the test definitions yourself anyway and than the investment isn't all that justified (though the BAs can still at least read and check the tests). On the other hand I can see a big marketing value towards business people - the sentence "You can write real test in Word" sounds really good to a lot of people so it may help sell an investment in specialized frameworks.

### SoapUI and other easy to use tools

SoapUI is a tool we used a lot in my previous company. It's great because you can define test in a UI which gives a low barrier for entry for testers and BAs without programming skills. It has two major downsides however:

1. No UI testing. Theoretically you can call Groovy scripts from SoapUI but it is very cumbersome especially if you need a lot of integration with other tests.
- More importantly the tests written in SoapUI are of the lowest abstraction. You are directly dealing with all the possible application details without any tools for encapsulating functionality. This means your tests will be of the worst kind mentioned in this article. So the low barrier to entry is actually just sucking you into an ugly mess when the breaking changes come... and they do come after some time.
- there is actually way more but this is not a rant against SoapUI maybe I'll write another post about that. would be fun. I like rants.

Other similar tools I could mention are GUI automation tools that record your clicks. Again no tools for encapsulation, your test will break and you'll have to fix them one by one.

### Links

- Continuous Delivery book: http://www.amazon.com/Continuous-Delivery-Deployment-Automation-Addison-Wesley/dp/0321601912
