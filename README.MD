![pgssoft-logo.png](pgssoft-logo.png)

# HttpClientMock

HttpClientMock is a library for mocking [Java HttpClient](https://docs.oracle.com/en/java/javase/11/docs/api/java.net.http/java/net/http/HttpClient.html). It has an intuitive fluent API for defining client behaviour and verifing number of made requests.

* [Installation](#installation)
* [Requirements](#requirements)
* [Usage](#usage)
* [Request matching](#request-matching)
* [Define response](#define-response)
* [Verification](#verification)
* [Examples](#examples)
* [License](#license)
* [About](#about)
* [Follow us](#follow-us)

## Installation

HttpClientMock is available in Maven Central Repository.
```
<dependency>
    <groupId>com.pgs-soft</groupId>
    <artifactId>HttpClientMock</artifactId>
    <version>1.0.0</version>
</dependency>
```

## Requirements

* Java 11

## Usage

#### Record
Working with HttpClientMock starts with defining client behaviour. Before code under tests starts HttpClientMock must know how to respond to every request.
```
HttpClientMock httpClientMock = new HttpClientMock();
httpClientMock.onGet("http://localhost/login")
  .withParameter("user","john")
  .doReturn("Ok");
httpClientMock.onPost("http://localhost/login").doReturnStatus(501);
```

#### Replay
Code under test starts and uses HttpClientMock with defined behaviour.
```
var get = HttpRequest.newBuilder(URI.create("http://localhost/login?user:john")).GET().build();
var post = HttpRequest.newBuilder(URI.create("http://localhost/login")).POST(noBody()).build();
httpClient.send(get, ofString()); // returns response with body "Ok"
httpClient.send(post, ofString()); // returns response with status 501
```

#### Verify
When code under test finishes, HttpClientMock allows to check number of made request. It is possible to use the same set of conditions as for defining mock behaviour.
```
httpClientMock.verify().get("http://localhost/login").withParameter("user","john").called()
httpClientMock.verify().post("http://localhost/login").notCalled()
```


## Request matching

### HTTP method
HttpClientMock supports all Http methods.
```
httpClientMock.onGet().doReturn("get");
httpClientMock.onPost().doReturn("post");
httpClientMock.onPut().doReturn("put");
httpClientMock.onDelete().doReturn("delete");
httpClientMock.onOptions().doReturn("options");
httpClientMock.onHead().doReturn("head");
```
### URL
Every `onGet()`, `onPost()`, .... method accept URL. It is possible to write:
```
httpClientMock.onGet("http://localhost/login?user=john").doReturnStatus(200);
```
which is equal to
```
httpClientMock.onGet()
  .withHost("httt://locahost")
  .withPath("/login")
  .withParameter("user","john")
  .doReturnStatus(200);
```

It is possible to define default host using HttpClientMock constructor, so later methods can accept relative URL-s.
```
HttpClientMock httpClientMock = new HttpClientMock("http://localhost");
httpClientMock.onGet("/login").doReturn("ok");
httpClientMock.onPost("/edit?user=john").doReturnStatus(200);

httpClientMock.onGet("http://www.google.com").doReturn("Google") // Absolute paths still work.
```

### Host, path, parameters, reference conditions
It is possible to define each part of url separately.
```
httpClientMock.onGet()
  .withHost("httt://locahost")
  .withPath("/login")
  .withParameter("user","john")
  .withReference("edit")
  .doReturnStatus(200);
```

### Header condition
```
httpClientMock.onGet("http://localhost/login")
  .withHeader("tracking","123")
  .doReturn("ok");
```

### Body condition
```
httpClientMock.onGet("http://localhost/login")
  .withBody("tracking",containsString(123))
  .doReturn("ok");
```

### Custom condition
```
Condition fooCondition = request -> request.getUri().contains("foo");
httpClientMock.onGet("http://localhost/foo/bar")
  .with(fooCondition)
  .doReturn("yes");
```

### Matchers
Every condition method accepts [Hamcrest Matcher](https://github.com/hamcrest/JavaHamcrest) which allows to define custom conditions on requests.
```
httpClientMock.onGet("http://localhost")
  .withPath(containsString("login"))
  .withParameter("user",equalToIgnoringCase("John)")
  .reference(not(equalTo("edit")));
```

### Multiple matching rules
If request matches more then one rule, then last defined one is used.

### None rule matche
If request doesn't matche any rule, HttpClientMock return response with status 404.

## Define response

### Response
Response with provided body and status 200.
```
httpClientMock.onGet("http://localhost").doReturn("my response")
```
### Status
Response with empty body and provided status
```
httpClientMock.onGet("http://localhost").doReturnStatus(300)
httpClientMock.onGet("http://localhost").doReturn("Overloaded").withStatus("500");
```
### Exception
Instead of returning response it throws defined exception.
```
httpClientMock.onGet("http://localhost").doThrowException(new IOException());
```
### Custom action
```
Action echo r -> {
  HttpEntity entity = ((HttpEntityEnclosingRequestBase) r.getHttpRequest()).getEntity();
  BasicHttpResponse response = new BasicHttpResponse(new ProtocolVersion("http", 1, 1), 200, "ok");
  response.setEntity(entity);
  return response;
};
httpClientMock.onGet("http://localhost").doAction(echo);
```
### Response header
```
httpClientMock.onPost("/login").doReturn("foo").withHeader("tracking", "123")
```
### Response status
```
httpClientMock.onPost("/login?user=bar").doReturn("Wrong user").withStatus(403)
```

### JSON
Response with provided body, status 200 and content type "application/json"
```
httpClientMock.onPost("/login").doReturnJSON("{foo:1}");
```

### XML
Response with provided body, status 200 and content type "application/xml"
```
httpClientMock.onPost("/login").doReturnXML("<foo>bar</foo>");
```

### Multiple actions
It is possible to add multiple actions to one rule. Every call will use next action until last is reached.
```
httpClientMock.onPut("/addUser")
  .doReturn("ok");
  .doReturnStatus(500);

var req = HttpRequest.newBuilder(URI.create("http://localhost/addUser")).PUT(noBody()).build();
httpClientMock.send(req, ofString()); //returns "ok"
httpClientMock.send(req, ofString()); //returns status 500
httpClientMock.send(req, ofString()); //returns status 500
```


## Verification
HttpClientMock allows to check how many calls were made. Verification supports the same set of conditions us rule defining.
```
httpClientMock.verify().get("http://localhost").called();

httpClientMock.verify().get("http://localhost/login")
  .withParameter("user","john")
  .called();

httpClientMock.verify().get("http://localhost/login")
  .withParameter("user","Ben")
  .notCalled();

httpClientMock.verify().delete().notCalled();

httpClientMock.verify().get().called(greaterThanOrEqualTo(1));

```
# Examples
## Example 1
```
// DEFINE BEHAVIOUR
HttpClientMock httpClientMock = new HttpClientMock("http://localhost:8080");
httpClientMock.onGet("/login?user=john").doReturnJSON("{permission:1}");
httpClientMock.onPost("/edit")
  .withParameter("user","John")
  .doReturn("ok")
  .doReturnStatus(503);

// EXECUTION
// request to http://localhost:8080/login?user=john returns JSON {permission:1}
// first request to http://localhost:8080/edit?user=john returns message "ok"
// second request to http://localhost:8080/edit?user=john returns request with status 503

// VERIFICATION
httpClientMock.verify().get("/login?user=john").called();
httpClientMock.verify().post("/edit?user=john").called(2);
httpClientMock.verify().delete().notCalled();
```


## Example 2
```
// DEFINE BEHAVIOUR
HttpClientMock httpClientMock = new HttpClientMock();
httpClientMock.onGet("http://localhost:8080/login").doReturn("Missing parameter user").withStatus(400);
httpClientMock.onGet("http://localhost:8080/login")
  .withParameter("user","JJohn")
  .doReturn("Wrong user name").withStatus(403);
httpClientMock.onGet("http://localhost:8080/login")
  .withParameter("user","John")
  .doReturn("ok");

// EXECUTION
// request to http://localhost:8080/login?user=john returns message "ok"

// VERIFICATION
httpClientMock.verify().get("/login?user=john").called();
```

## License

The project is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).

## About

The project maintained by [software development agency](https://www.pgs-soft.com/) [PGS Software](https://www.pgs-soft.com/).
See our other [open-source projects](https://github.com/PGSSoft) or [contact us](https://www.pgs-soft.com/contact-us/) to develop your product.

## Follow us

[![Twitter Follow](https://img.shields.io/twitter/follow/pgssoftware.svg?style=social&label=Follow)](https://twitter.com/pgssoftware)

