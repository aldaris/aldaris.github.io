---
layout: post
title:  "Hunting down a bug in Tomcat"
date:   2017-08-25 13:52:00 +0100
categories: dev
---
Not too long ago I was asked to look into a strange bug, [OPENAM-11571](https://bugster.forgerock.org/jira/browse/OPENAM-11571). Although the reproduction of the issue wasn't too simple (issues with our home-grown test framework used for performance tests), the actual debugging process turned out to be quite something. In this post I'll try to document my thought process, and the steps I taken to figure out what's going on.

The issue itself was SAML related and it all started with this stacktrace:

```
Stacktrace:
  org.apache.jasper.servlet.JspServletWrapper.handleJspException(JspServletWrapper.java:579)
  org.apache.jasper.servlet.JspServletWrapper.service(JspServletWrapper.java:471)
  org.apache.jasper.servlet.JspServlet.serviceJspFile(JspServlet.java:396)
  org.apache.jasper.servlet.JspServlet.service(JspServlet.java:340)
  javax.servlet.http.HttpServlet.service(HttpServlet.java:729)
  org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
  </pre><p><b>root cause</b></p><pre>java.lang.IllegalStateException: Cannot forward after response has been committed
  com.sun.identity.saml.common.SAMLUtils.forwardRequest(SAMLUtils.java:1812)
  com.sun.identity.saml.common.SAMLUtils.sendError(SAMLUtils.java:1766)
```

The *response has been committed* error usually just means that there was an attempt to change something on the response, when we already sent back something to the caller on a different codepath.

Since the issue seemed to happen in the *sendError* method, my assumption was that probably an error occurred in the middle of processing a request, which would explain why the response was already (half?) committed.

After adding a few debug statements to the surrounding code and having a look at Wireshark network dumps, it turned out that the fedlet failed to handle an error properly that it had received from the SAML IdP. Further investigations then revealed that the IdP was returning the errors incorrectly (sending HTML content rather than SOAP faults)... The logs on the SAML IdP side were a bit cryptic:

```
ERROR: IDPArtifactResolution.doArtifactResolution: SOAP error
com.sun.xml.messaging.saaj.SOAPExceptionImpl: Unable to internalize message
  at com.sun.xml.messaging.saaj.soap.MessageImpl.init(MessageImpl.java:536)
  at com.sun.xml.messaging.saaj.soap.MessageImpl.<init>(MessageImpl.java:316)
  at com.sun.xml.messaging.saaj.soap.ver1_1.Message1_1Impl.<init>(Message1_1Impl.java:80)
  at com.sun.xml.messaging.saaj.soap.ver1_1.SOAPMessageFactory1_1Impl.createMessage(SOAPMessageFactory1_1Impl.java:78)
  at com.sun.identity.saml2.profile.IDPArtifactResolution.doArtifactResolution(IDPArtifactResolution.java:199)
  at com.sun.identity.saml2.servlet.IDPArtifactResolutionServiceSOAP.doPost(IDPArtifactResolutionServiceSOAP.java:53)
  at javax.servlet.http.HttpServlet.service(HttpServlet.java:648)
  at javax.servlet.http.HttpServlet.service(HttpServlet.java:729)

com.sun.xml.messaging.saaj.SOAPExceptionImpl: Error setting the source for SOAPPart: null
  at com.sun.xml.messaging.saaj.soap.SOAPPartImpl.setContent(SOAPPartImpl.java:281)
  at com.sun.xml.messaging.saaj.soap.MessageImpl.init(MessageImpl.java:407)
  at com.sun.xml.messaging.saaj.soap.MessageImpl.<init>(MessageImpl.java:316)
  at com.sun.xml.messaging.saaj.soap.ver1_1.Message1_1Impl.<init>(Message1_1Impl.java:80)
  at com.sun.xml.messaging.saaj.soap.ver1_1.SOAPMessageFactory1_1Impl.createMessage(SOAPMessageFactory1_1Impl.java:78)
  at com.sun.identity.saml2.profile.IDPArtifactResolution.doArtifactResolution(IDPArtifactResolution.java:199)
  at com.sun.identity.saml2.servlet.IDPArtifactResolutionServiceSOAP.doPost(IDPArtifactResolutionServiceSOAP.java:53)
  at javax.servlet.http.HttpServlet.service(HttpServlet.java:648)
```

Given that this is not an error message that I normally see, I was a bit confused, and I tried to figure out what this error actually means, and whether it's to do with how OpenAM is using the SAAJ APIs (responsible to process and create SOAP messages).

Fortunately with the help of IntelliJ I was able to debug this a bit further, and found that there is something fishy going on: even though SAAJ was unable to read the complete SOAP message, I could easily convert the HTTP request's payload to a string right after a failure.

This made me wonder whether there is any chance that the SAAJ APIs aren't thread-safe (OpenAM stores MessageFactory instances in a static field). Given that this issue only seems to happen during high load, this didn't seem like a daft idea at the beginning. After looking around on [StackOverFlow](https://stackoverflow.com/questions/25279280/javax-xml-soap-messagefactorys-instance-is-thread-safe) and browsing the [MessageFactory JavaDocs](https://docs.oracle.com/javase/8/docs/api/javax/xml/soap/MessageFactory.html) there seemed to be no official stance on whether MessageFactory is thread-safe, so I started to look at its source in IntelliJ, and after some careful look-around I've determined that the SAAJ implementation used by OpenAM is thread-safe.

Well that theory is thrown out of the window then, but at least during this code reading exercise I have realized that there is an additional error message logged by SAAJ to the standard error output, that will end up in catalina.out log file on Tomcat. Maybe next time I should look at all the log files first...

The stacktrace in catalina.out looked like this:

```
java.net.SocketTimeoutException
  at org.apache.tomcat.util.net.NioBlockingSelector.read(NioBlockingSelector.java:202)
  at org.apache.tomcat.util.net.NioSelectorPool.read(NioSelectorPool.java:250)
  at org.apache.tomcat.util.net.NioSelectorPool.read(NioSelectorPool.java:231)
  at org.apache.coyote.http11.InternalNioInputBuffer.fill(InternalNioInputBuffer.java:133)
  at org.apache.coyote.http11.InternalNioInputBuffer$SocketInputBuffer.doRead(InternalNioInputBuffer.java:177)
  at org.apache.coyote.http11.filters.IdentityInputFilter.doRead(IdentityInputFilter.java:110)
  at org.apache.coyote.http11.AbstractInputBuffer.doRead(AbstractInputBuffer.java:362)
  at org.apache.coyote.Request.doRead(Request.java:476)
  at org.apache.catalina.connector.InputBuffer.realReadBytes(InputBuffer.java:350)
  at org.apache.tomcat.util.buf.ByteChunk.substract(ByteChunk.java:395)
  at org.apache.catalina.connector.InputBuffer.read(InputBuffer.java:375)
  at org.apache.catalina.connector.CoyoteInputStream.read(CoyoteInputStream.java:190)
  at com.sun.xml.messaging.saaj.util.ByteOutputStream.write(ByteOutputStream.java:92)
  at com.sun.xml.messaging.saaj.util.JAXMStreamSource.<init>(JAXMStreamSource.java:66)
  at com.sun.xml.messaging.saaj.soap.SOAPPartImpl.setContent(SOAPPartImpl.java:245)
```

Now this is a bit peculiar, there was a read timeout whilst reading the incoming request's InputStream? Since this was a very strange error, I have started to download the Tomcat source code and loaded it into IntelliJ. In case you'll need to do this in the future:

* there are git repositories mirroring the official SVN repositories for Tomcat,
* each version has its own repository on GitHub, like [this](https://github.com/apache/tomcat70), [this](https://github.com/apache/tomcat80) and [this](https://github.com/apache/tomcat85),
* you will probably need to read the BUILDING.txt file in the repository to figure out the minimum version of [ant](https://ant.apache.org/).

Heading straight to the NioBlockingSelector class, this code will be of interest:

```java
while(!timedout) {
...
  if (readTimeout >= 0 && keycount == 0)
    timedout = (System.currentTimeMillis() - time) >= readTimeout;
} //while
if (timedout)
  throw new SocketTimeoutException();
```

Knowing that the stacktrace came exactly from the last line, I started to look into why *timedout* could become *true*. The only assignment to the variable was in the last *if* statement of the *while* block, so that condition must be true for this bug to happen... This means that *readTimeout* must be greater than or equal to 0 and *keycount* must be 0. Now I have still very little understanding of what *keycount* does, but the fact that *readTimeout* was >= 0 meant that maybe there is a timeout value set somewhere that isn't great enough. I tend to be optimistic about these sort of things, so at this point I was really just looking forward to find a magic setting that controls the read timeout value that I can increase and document. Once I attached the debugger, I put a breakpoint to the throw statement and triggered the issue with the performance test again.

Once the breakpoint was hit I could see a couple of things:

* the condition for setting the *keycount* variable to 0 was never true by the time the exception was thrown,
* *readTimeout* was set to 0

According to the method's JavaDoc:

```
* @param readTimeout long - the timeout for this read operation in milliseconds, -1 means no timeout
```

Well, it looks like that a very unrealistic 0 ms timeout is being enforced on this socket, so let's have a look at how this method is invoked and where the *readTimeout* value is coming from:

```java
nRead = pool.read(readBuffer, socket, selector, socket.getIOChannel().socket().getSoTimeout());
```

The *socket()* method returns a *java.net.Socket* instance, so let's have a look at the [JavaDoc](https://docs.oracle.com/javase/8/docs/api/java/net/Socket.html#getSoTimeout--) of the *getSoTimeout* method:

```
* Returns setting for {@link SocketOptions#SO_TIMEOUT SO_TIMEOUT}.
* 0 returns implies that the option is disabled (i.e., timeout of infinity).
```

Wait a minute... So 0 for *Socket* means no read timeout, but for *NioBlockingSelector* only -1 means the same thing? Looks like I've found a bug then. :)

Resolving bugs is never as straightforward as one may think, for example there is a very strong urge to change the last *if* statement in the *while* block to this:

```java
if (readTimeout > 0 && keycount == 0)
  timedout = (System.currentTimeMillis() - time) >= readTimeout;
```

The couple of years I've spent fixing bugs in OpenAM taught me that there is usually more to these issues than expected. Looking at the [git blame](https://github.com/apache/tomcat80/blame/6544e08347aeabed26214d9cfdb480d7a627a1d9/java/org/apache/tomcat/util/net/NioBlockingSelector.java) of the line in question revealed that the *if* statement was changed with [this commit](https://github.com/apache/tomcat80/commit/00eb265f35e9a11c6d485cade7c41c39221dfada). Right, so without that change the while loop just kept on going and potentially using high CPU throughout the process.

My next thought was to try to get some control over the *keycount* variable and prevent that condition being triggered. The relevant code snippet for this would be:

```java
try {
  if (att.getReadLatch() == null || att.getReadLatch().getCount() == 0)
    att.startReadLatch(1);
  poller.add(att, SelectionKey.OP_READ, reference);
  if (readTimeout < 0) {
    att.awaitReadLatch(Long.MAX_VALUE, TimeUnit.MILLISECONDS);
  } else {
    att.awaitReadLatch(readTimeout, TimeUnit.MILLISECONDS);
  }
} catch (InterruptedException ignore) {
    // Ignore
}
if (att.getReadLatch() != null && att.getReadLatch().getCount() > 0) {
  //we got interrupted, but we haven't received notification from the poller.
  keycount = 0;
} else {
  //latch countdown has happened
  keycount = 1;
  att.resetReadLatch();
}
```

Reading this through you'll see that there is an *if* statement in there checking the *readTimeout* value, and looks like depending on the value of it we either wait for *Long.MAX_VALUE* or only for *readTimeout* interval. This is very promising! Changing the *if* condition to check if the *readTimeout* variable is less than or equal to 0 should trigger a semi-infinite wait, and as such should prevent the SocketTimeoutException as well. You can find my PR for this change on [GitHub](https://github.com/apache/tomcat80/pull/9).

A different solution for this would be to change the invoking code so that it converts 0 *Socket#getSoTimeout* values to -1. Given that the latest 8.5.x version is not affected by this bug and that's mainly because a [fix just like that](https://github.com/apache/tomcat85/blob/trunk/java/org/apache/tomcat/util/net/SocketWrapperBase.java#L140) was implemented for it, there is a good chance that my PR won't be taken in as is, but time will tell. Either case, I found a bug in Apache Tomcat, achievement unlocked. :)

### Takeaway

My key takeaway from all of this is that I wouldn't want to use Tomcat 7.0.15+ and Tomcat 8.0.x versions in production environments, because it can randomly fail to handle incoming requests.

#### UPDATE

The bug is now fixed in both Tomcat [7.0.x](http://svn.apache.org/viewvc?view=revision&revision=1806734) and [8.0.x](http://svn.apache.org/viewvc?view=revision&revision=1806733) and should make its way into Tomcat 7.0.82 and Tomcat 8.0.47.
