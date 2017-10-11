---
layout: post
title:  "Deep dive into Cross Site Request Forgery"
date:   2017-10-11 09:27:00 +0100
categories: dev security
---
I've been always fascinated by Cross Site Request Forgery attacks, so today I'm going to try to go through the different variants of this attack type in as much level of detail as I can. I will also try to give some guidance on how to prevent these attacks depending on the attack surface.

### Disclaimer

I really don't believe that I'm an expert in online security. The things I detail below reflects my current understanding of the subject, and as such should be taken with a pinch of salt. I would definitely recommend to read other online resources on this subject before drawing any conclusions.

Without further ado, let's get into it.

### An example CSRF attack

For demonstration purposes, let us assume that there is a very simple (but quite vulnerable) application that has the following HTML snippet on one of its pages:

```html
<form action="/update" method="post">
  <input type="text" name="username" />
  <input type="password" name="password" />
  <input type="submit" value="Submit" />
</form>
```

The server side code for processing these requests looks like the following:

```java
public void service(ServletRequest req, ServletResponse res) {
    checkSessionIsAuthorized(req.getCookies());
    db.updatePassword(req.getParameter("username"), req.getParameter("password"));
}
```

If you look carefully, you'll notice that the server side does not check whether the request method was **POST**, it just blindly uses the parameters from the request to perform the operation. With this knowledge in mind let's consider the scenario when an attacker sets up a page with the following snippet:

```html
<img src="http://example.com/update?username=demo&password=pwned" height="1" width="1" />
```

The attacker then sends a harmless looking link to the victim, and waits for that link to get opened. When the victim clicks on the link, the "image" will be rendered by their browser and as part of this rendering process the browser will include the session cookies for *example.com*. This means that it will silently perform the update operation on behalf of the victim, thus changing demo user's password to *pwned* (assuming the victim actually had the necessary privileges).

There are two important things that I need to point out here:

* Having an application that performs state changes via a **GET** request is [not recommended][HTTP methods], and it also makes your application a lot more insecure.
* The embedded content does not have to be hosted by the attacker, it could be hosted on a site which allows embedding external content, or that is vulnerable to [Content Injection] vulnerability.

### Example CSRF attack using POST

For this example the application has been updated to reject requests with **GET** request method, and only perform the update operation when the request method is **POST**:

```java
public void doGet(HttpServletRequest request, HttpServletResponse response)
        throws ServletException {
    throw new ServletException("Go away");
}

public void doPost(HttpServletRequest req, HttpServletResponse resp) {
    checkSessionIsAuthorized(req.getCookies());
    db.updatePassword(req.getParameter("username"), req.getParameter("password"));
}
```

Whilst this change makes sure that the request cannot be performed using the **GET** method, the request can still be made using **POST** method. The easiest way to trigger a POST request to a page is to submit a form, for example this one:

```html
<form action="http://example.com/update" method="POST">
  <input type="text" name="username" value="demo" />
  <input type="text" name="password" value="pwned" />
</form>
```

The first thing to mention here is that this form can be literally on any website. It is perfectly legal to render a form on *domain1* and submit its contents to *domain2*.

With the usage of hidden input fields this attack can be improved even further. When the unsuspecting victim visits the page, the form will be rendered with invisible input fields, and with an extra bit of JavaScript code the form could be automatically submitted to the vulnerable site. The end result is that the request method is **POST** and all the required parameters are in the request payload as expected. A complete exploit would be:

```html
<html>
  <head><title>I'm not an attacker, honest</title></head>
  <body onload="document.forms[0].submit()">
    <form action="http://example.com/update" method="POST">
      <input type="hidden" name="username" value="demo" />
      <input type="hidden" name="password" value="pwned" />
    </form>
  </body>
</html>
```

The main similarity between these CSRF examples is that the attacker does not see the HTTP response to these forged requests, however they don't actually have to. As long as they can monitor the access logs for the exploit pages (if they host them themselves), or simply test whether the intended request was actually made (for example by trying to authenticate the *demo* user with the *pwned* password), they don't really need to actually see the response to these requests.

Another thing to note is that the first example left the user oblivious that the forged request was made, given that only a 1x1 pixel image failed to display. In case of the auto-submitting form example, the user will be directed away to the page (top level navigation), and they will see a suspicious "update successful" message. One way to get around this would be to render the auto-submitting form within an invisible frame/iframe instead. To prevent this, the recommendation is to either set the [X-Frame-Options] or the [Content-Security-Policy] header (with [frame-ancestors][CSP frame ancestors] whitelist) on the application side, as this will instruct browsers to not render the contents of the site within frames.

### Referer header validation

Given that the Referer (sic) header identifies the address of the webpage that linked to the resource being requested, it is possible to use the Referer header for CSRF prevention. An application can define a whitelist of URLs/domains that it will always accept, and if there is any other value provided in the Referer header, the request can be rejected. The advantage of this implementation is that it doesn't require server side state and can be implemented relatively easily. The downside of this approach is that the Referer header is not a mandatory header, and there are times when it is not sent at all (depends on [Referrer policy] for example).

### Origin header validation

The Origin header indicates where a request is originated from, but only discloses the protocol, hostname, and the port number. Similarly to the Referer header, the Origin header value can be used to prevent CSRF attacks with the same whitelist approach. The problem is that the Origin header isn't always included in requests either.

The recommendation according to [OWASP] is to implement both Referer and Origin header validation, and reject requests that has neither of the headers.

### CSRF tokens to the rescue

Whilst the header whitelisting is definitely a solution that can work, maintaining the whitelists can be a bit painful at times. Most of the time applications aren't really bothered by storing a bit of extra data on the server side, hence the most commonly implemented protection against CSRF is the use of CSRF tokens. The essence of this method is that each HTML form in the application is protected by a random (**CSRF**) token. These CSRF tokens are part of the forms as hidden input fields so that they are going to be submitted along with the rest of the form fields. When the form is submitted, the CSRF token is verified on the server side.

The pseudo code for this would look something like:

```java
public void doGet(HttpServletRequest request, HttpServletResponse response) {
    token = generateRandomId
    store token on server side
    embed token into HTML response as a hidden input field
}

public void doPost(HttpServletRequest request, HttpServletResponse response) {
    read token from request
    if token is same as token stored on server side
      process request and perform operation
    else
      display error and stop processing request
}
```

The beauty of this solution is that for a successful exploit the attacker has to include the correct CSRF token in the request, however that should not be possible (assuming that the CRSF tokens are well guarded and are generated using a secure pseudo-random number generator and are tied in one way or another to the logged in user).

Given that random tokens provide a pretty solid protection against CSRF attacks, any kind of an attack that can steal CSRF tokens from a victim has increased value. One very interesting way to steal CSRF tokens is to perform [Path Relative StyleSheet Import] attack for example. Just a friendly reminder that security is a never-ending task...

### CSRF via AJAX

First and foremost we have to learn about the [same-origin policy]: this is a built-in security feature of browsers that **restricts how a document or script loaded from one origin can interact with a resource from another origin**, where *origin* represents a scheme/host/port tuple. One aspect of the policy is that a page under *http://evil.com/index.html* for example has no permission to send requests to *http://example.com/update*, because the origins are different (*http://evil.com:80* vs *http://example.com:80*).

What all of this means is that if the attacker directs an end-user to *evil.com*, they will not be able to make AJAX requests on behalf of the user to *example.com* under normal circumstances. Given that the same-origin policy can be found too restrictive at times, there is a separate feature which allows exceptions from it, called [CORS] (Cross Origin Resource Sharing). The CORS standard defines a set of HTTP headers that can be set by applications to allow AJAX requests from different origins.

Let us assume the worst possible case where the application in question sets the following CORS headers on all of its pages:

```html
Access-Control-Allow-Origin: http://evil.com
Access-Control-Allow-Credentials: true
```

This means that the application's pages are accessible via AJAX from the *evil.com* domain, and the credentials are also made available during the AJAX requests. In this scenario the attacker could embed the following JavaScript code into a page on *evil.com*:

```javascript
$(document).ready(function() {
    $.ajaxSetup({crossDomain: true, xhrFields: {withCredentials: true}});
    $.get('http://example.com/update').done(function(data) {
        var csrf = data.match(/.*name="csrf"[^>]*value="([^"]*)".*/)[1];
        $.ajax({
            type: "POST",
            url: "http://example.com/update",
            data: "username=demo&password=pwned&csrf=" + csrf
        }).done(function(data) {
            console.log("Hack successful");
        });
    });
});
```

The gist of this exploit is that it first sends a request as the end-user to the update page to obtain the CSRF token from the HTML response. Once this is complete, there is a second request made that includes the CSRF token and the other input fields to actually execute the operation. A couple of things I should highlight about this attack:

* I was only able to perform the operation, because of the **Access-Control-Allow-Credentials** header. Without it, the AJAX call would not have included the end-user's cookies, and as such the call would have been unauthenticated.
* Fortunately the CORS spec is very sensible when it comes to **Access-Control-Allow-Credentials**, it cannot be used along with **Access-Control-Allow-Origin: \***, this should prevent misconfigurations exposing applications to these sort of attacks.
* This kind of attack would just as well work if script can be injected into the same origin as the application (and then you wouldn't need CORS headers either).
* I've used *evil.com* in the example, but the reality is that any site trusted via CORS can become evil if they are not protected against Content Injection attacks.

### Protecting REST APIs from CSRF attacks

It is quite common to leave REST APIs accessible to the end-users, either because they are being used by JavaScript based user interfaces, or to expose the application's resources to external clients. The main difference between the REST APIs and regular HTML pages is that browsers tend to retrieve the pages before trying to submit them, hence it is easy to make the CSRF token available to them. When it comes to REST APIs, quite often this is not the case, the API calls are most of the time one-off requests, which makes this approach rather difficult:

* If the CSRF token is the same or a derived/computed value from the session ID, the problem is that the session ID optimally should not be available for JavaScript applications -- for example because the cookie is [HttpOnly].
* If the CSRF token is an encrypted token, then there needs to be a way for the client to retrieve this value before actually using it. This however requires a reasonable mechanism to distribute the encrypted token (as part of an authentication response for example), but one cannot always guarantee that the end-user have used the application in question to authenticate. Forcing API users to send two requests per operation (one for getting a CSRF token, one for actually performing the operation) is probably not going to scale well.

An alternative solution to CSRF tokens is the usage of custom headers, however those have their pros and cons as well:

* Requiring a custom header on the REST endpoint deals with simple browser requests (think of the first and second CSRF examples), but requires [additional CORS setup][Access-Control-Allow-Headers] when working with AJAX clients.
* If an origin is whitelisted in CORS to make REST calls, then the application is only protected from CSRF attacks as long as the trusted origin is not vulnerable to Content Injection attacks.

I should probably also highlight that restricting which HTTP verbs can be used by the REST clients is not an adequate protection either: various [frameworks][symfony method override] or [containers][nodejs method override] even allow the modification of the request method based on request parameter/header.

So all in all, when it comes to REST APIs, implementing CSRF protections is not really that straightforward and requires careful design process. Combining one of the above approaches with Origin/Referer header validation can provide a decent level of protection against CSRF attacks at REST APIs, however I've got to say that sounds like a lot of work...

### Future of CSRF attacks

Fortunately CSRF doesn't have a long term future, the currently draft spec of [SameSite] cookies will prevent these attacks completely. The new SameSite flag on cookies instructs the browser to only include the cookie values when the top level navigation origin matches the cookie's domain. The support for SameSite cookies is still [work in progress][caniuse], but hopefully it will be widely adopted soon. Whilst it will prevent CSRF attacks altogether, chances are that there will be new sort of attacks surfacing in its place.

### Other types of CSRF attacks

All the previously mentioned CSRF attacks assumed that the operation to be performed was something nasty, like changing password, or elevating user privileges. It is also possible to use CSRF to login or logout even.

In case of login CSRF you can invade the victim's privacy by letting them authenticate as the attacker and then use the application as they would normally do (think of browsing history stored within the app). Logout CSRF on the other hand can be used to silently log out the end-user from an application and hence force them to re-authenticate.

### Takeaway

Cross Site Request Forgery attacks are quite tricky to deal with, but there are several well established methods to prevent them fortunately. In my opinion the following best practices can help one secure their applications:

* To limit content injection, disable [Content Type Sniffing] in your application.
* Use either [X-Frame-Options] of [Content-Security-Policy] headers with frame-ancestors.
* To limit Cross Site Scripting attacks that could be used for content injection or stealing CSRF tokens, set [X-XSS-Protection] header.
* Use [HttpOnly], [Secure], and [SameSite] cookies wherever you can, they really make your applications more secure.
* Review your CORS setup and try to only set the headers for the resources that you actually want to make available cross domain. Try to separate your web frontend from backend APIs to allow more fine-tuned CORS configurations.
* Use CSRF tokens in your forms and consider implementing Origin/Referer header validation for your REST endpoints.

Most of the above are actually very easy to do (see also [securityheaders.io]), but to really protect against CSRF attacks (especially when implementing REST APIs) one really has to consider the different use-cases for their application. This is why I would strongly recommend to consider CSRF prevention as early in the development phase as possible.

[HTTP methods]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods
[Content Injection]: https://www.owasp.org/index.php/Content_Spoofing
[X-Frame-Options]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
[Content-Security-Policy]: https://content-security-policy.com/
[CSP frame ancestors]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/frame-ancestors
[Referrer policy]: https://www.w3.org/TR/referrer-policy/
[OWASP]: https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29_Prevention_Cheat_Sheet
[Path Relative StyleSheet Import]: http://blog.portswigger.net/2015/02/prssi.html
[same-origin policy]: https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy
[CORS]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS
[HttpOnly]: https://www.owasp.org/index.php/HttpOnly
[Secure]: https://en.wikipedia.org/wiki/Secure_cookies
[symfony method override]: https://stackoverflow.com/questions/26936163/override-the-http-method-for-get-requests-in-symfony2
[nodejs method override]: https://github.com/expressjs/method-override
[Access-Control-Allow-Headers]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Headers
[SameSite]: https://tools.ietf.org/html/draft-west-first-party-cookies-07
[caniuse]: https://caniuse.com/#search=samesite
[Content Type Sniffing]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options
[X-XSS-Protection]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection
[securityheaders.io]: https://securityheaders.io/
