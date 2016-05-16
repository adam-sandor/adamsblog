+++
date = "2016-05-12T12:42:04+02:00"
draft = true
title = "Why you should build your own test framework"
+++

I have been a big fan of test automation ever since I first tried it and thats no big thing. These days nobody denies the need for tests that can be executed quickly and frequently to support the short iterations of agile development. What I still see often however is test code that isn't given the same care as production code because.. well.. it's just test code right? It's true that badly written test code won't cause a production outage but if you are serious about your testing and write a lot of tests the test code can become a huge maintenance burden. I'm going to focus on the upper part of the testing pyramid - tests that are interchangeably called acceptance or functional tests.

A typical application would be tested using it's UI and some exposed webservice APIs. The straightforward approach to testing is to use the test frameworks available to us and write some tests using them. Here I'll be using [Groovy HttpBuilder] to submit an order to a webshop's API webservice and [Geb](http://www.gebish.org) to verify if the order has appeared in the admin UI. I don't try to make this actually working code but I also didn't want to go pseudocode all the way.

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

### One step further

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

You can see that I explicitly specified the packages for the ApplicationDriver and the Order classes. This is to show a lesson I learned during implementing such test frameworks/DSLs. You shouldn't reuse classes from the production code base. It not surprising that ApplicationDriver is in the myshop.test package but there probably already is a class for representing Orders

### The stack



![Test application stack](/images/teststack.png)

### Made for testers

Another important aspect is the simplification of the test itself. While for the previous implementation needed knowledge of two frameworks the new only requires knowledge of the application specific DSL. This DSL will by nature be quite limited so learning it should be straightforward. This means testers with less programming knowledge will be able to write tests using the framework. This can be very useful for a lot of organizations. Having access to such a tool that is programming but in a simplified way can be a nice stepping stone for testers coming from a non-technical background but interested in test automation.

### Complexity of implementation

Flexibility

### BDD Frameworks

### SoapUI

### Links

- Continuous Delivery book: http://www.amazon.com/Continuous-Delivery-Deployment-Automation-Addison-Wesley/dp/0321601912
