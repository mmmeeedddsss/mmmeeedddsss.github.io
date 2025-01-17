---
title: TIL - You need to close responses returned from Apache's HttpConnection
description: Java's ClosableHttpResponse objects need to be closed to prevent resource leaks, which is very counter-intuitive.
date: 2024-02-07 15:35 +0300
categories: [CS, Java]
tags: [java, httpconnection, resource-leak]     # TAG names should always be lowercase
---

Imagine reviewing the following Java code:

```java
CloseableHttpClient httpClient = HttpClientBuilder.create()
    ...
    .build();


// Say that you have a list of objects to send to a server via post requests
// So you divide it into smaller chunks and send the requests in a loop
for(int i=0; i<50; i++) {
    try {
        HttpPost post = new HttpPost("https://<Server Url>");
        post.addHeader(HttpHeaders.CONTENT_TYPE, "application/json");
        post.setEntity(new StringEntity(getChunk(i)));

        HttpResponse resp = httpClient.execute(post);

        System.out.println("Sent " + i + "th chunk with status code " + resp.getStatusLine().getStatusCode());
    }
    catch (Exception ex) {
        System.out.println("Got an exception: " + ex.getMessage());
    }
}

```

Would you have any comments? (Other than the weird looking print statements and hard-coded server url etc. :smile:)

What about the following Python code?

```python
import requests

for i in range(50):
    try:
        response = requests.post("https://<Server Url>", json=chunk(i))
        print(f"Sent {i}th chunk with response code: {response.status_code}")
    except Exception as e:
        print(f"Failed to send {i}th chunk with exception: {e}")
```

Would you, again, have any comments?

---

Interestingly, what I learned the hard way is, there is a critical bug in the Java code above,
whereas 1 to 1 mapped Python code is perfectly fine.

Running the Java code above, we got the following output:

```
Sent 0th chunk with status code 200
Sent 1th chunk with status code 200
Retrying request on retry handler. Retry count: 1
Feb 07, 2024 3:38:01 PM org.apache.http.impl.execchain.RetryExec execute
INFO: I/O exception (org.apache.http.conn.ConnectionPoolTimeoutException) caught when processing request to {s}-><Server URL>:443: Timeout waiting for connection from pool
Feb 07, 2024 3:38:01 PM org.apache.http.impl.execchain.RetryExec execute
INFO: Retrying request to {s}-><Server URL>:443
Retrying request on retry handler. Retry count: 2
Feb 07, 2024 3:38:03 PM org.apache.http.impl.execchain.RetryExec execute
INFO: I/O exception (org.apache.http.conn.ConnectionPoolTimeoutException) caught when processing request to {s}-><Server URL>:443: Timeout waiting for connection from pool
Feb 07, 2024 3:38:03 PM org.apache.http.impl.execchain.RetryExec execute
INFO: Retrying request to {s}-><Server URL>:443
Retrying request on retry handler. Retry count: 3
Feb 07, 2024 3:38:05 PM org.apache.http.impl.execchain.RetryExec execute
INFO: I/O exception (org.apache.http.conn.ConnectionPoolTimeoutException) caught when processing request to {s}-><Server URL>:443: Timeout waiting for connection from pool
Feb 07, 2024 3:38:05 PM org.apache.http.impl.execchain.RetryExec execute
INFO: Retrying request to {s}-><Server URL>:443
Got an exception: Timeout waiting for connection from pool
```

The error we get is `Timeout waiting for connection from pool`.
It looks like a timeout while connecting to server since we are very used to these errors, but
actually, it is not such an error indicating a connection issue with the server; it indicates that
we cannot get an idle connection from the internal connection pool of `HttpClient` class.
The default pool size of the connection pool being 2, the 3rd request we sent failed since
the existing connection instances in the pool are busy, but why??

It appears that the response instance returned in the line
```java
HttpResponse resp = httpClient.execute(post);
```
has an `InputStream` attached and needs to get closed when you are done with it.

Even if you don't care about the returned body, as in our example, not caring about it causes a resource leak.

So to handle the HttpResponses well, you need to do one of the following in Java:
- Closing the response if you have to have a `ClosableHttpResponse` instance. In that case, you can just call `resp.close()`
- Close the `InputStream` on the response. A convenient way of doing that seems like `EntityUtils.consumeQuietly(resp.getEntity)`.
The implementation of `consumeQuietly` is very safe, being in try-catch and checking for nulls before anything.
- Java has a syntax named try-with-resources, which I understand, is similar to Python's `with` statement.
Using this syntax, you can auto close such closable resources when you are done with them. In our case, the following
syntax should also be fine:
  ```java
  try(final HttpResponse resp = httpClient.execute(post);) {
      System.out.println("Sent " + i + "th chunk with status code " + resp.getStatusLine().getStatusCode());
  }
  ```

---

I was talking with my dear teammate about the case here and, we think the same on the expected
behaviour of `HttpResponse resp = httpClient.execute(post);` should have been isolating all the
IO stuff from the developer and return a complete and finalized `HttpResponse` instance.
Not knowing Java that well, I think the selected implementation style here is unexpected.
For example, in the Python implementation, which I love more, the response object is final and does not
require/expect you to call on anything on it. The operation we do feels higher level than
sockets or kernel level stuff. I would expect to get a *connection*, or a *socket*,
or maybe a *file handler* returned if the implementation wants me to handle the
deallocation/freeing of resources.


I also fail to see any disadvantages of how Python handles the same task: Would it be
harder for Python to receive a continuous stream of data with its current way of implementation?
I think not, especially for HTTP.

So, we can conclude that Python is better :upside_down_face:.

---

**tldr**: Apache's HttpClient returns a `HttpResponse` object for responses, and
you need to close either the `InputStream` or the `HttpResponse` to allow freeing the resources.
Need for closing `HttpResponses` returned by `HttpConnections` on Java was very unintuitive for me xd

*Most of the information I collected in this investigation roots from the following stack overflow link: [https://stackoverflow.com/questions/11875015/httpclient-exception-org-apache-http-conn-connectionpooltimeoutexception-timeo](https://stackoverflow.com/questions/11875015/httpclient-exception-org-apache-http-conn-connectionpooltimeoutexception-timeo)*
