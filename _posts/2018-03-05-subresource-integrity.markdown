---
layout: post
title:  "Subresource integrity"
date:   2018-03-05 07:00:00 +0000
categories: dev security
---
Just this last month there was a slightly disturbing story unfolding: Browsealoud, a commonly used accessibility service (also used by several government websites) has been hacked and JavaScript code was injected into their service. The story itself can be found [here][motherboard], but if you want to read the security researcher's view of the subject, then I would suggest to check out [Scott Helme]'s and [Troy Hunt]'s excellent summaries as well.

The common takeaway from Scott and Troy appears to be that [Content Security Policy][CSP] (CSP) and [Subresource Integrity] (SRI) would have prevented this hack from happening. In the following I will try to share as much information as I can about SRI and how it can be leveraged via CSP, and describe how these technologies can be used to prevent issues like above.

### The basic concept of SRI

SRI allows one to declare their JavaScript and CSS references with an additional `integrity` attribute that holds the cryptographic hash of the file referenced. For example:

```html
<script
  src="https://code.jquery.com/jquery-3.3.1.js"
  integrity="sha256-2Kok7MbOyxpgUVvAk/HJ2jigOSYS2auK4Pfzbm7uH60="
  crossorigin="anonymous"></script>
<link
  rel="stylesheet"
  href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css"
  integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm"
  crossorigin="anonymous">
```

If the referenced JavaScript or CSS file has a different hash than what can be found in the integrity attribute, the downloaded resource will be discarded by the browser. The value of the integrity attribute is a space separated list of **algorithm-hash** combinations where the currently accepted algorithms are: **sha256**, **sha384**, and **sha512**.

The validation process is pretty well defined in the spec fortunately, but here is the gist of it:

* The browsers should pick the strongest algorithm they support from the listed values.
* If there are multiple hashes with the same algorithm, the validation will succeed as long as one of the values matches.

### SRI validation and same-origin policy

Let's imagine that there is an attacker who has the following HTML snippet on their site:

```html
<script
  src="https://internal.server/sensitive-data"
  integrity="sha256-5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8">
</script>
```

When a victim visits this site, their browser would attempt to verify the integrity of the resource, but the outcome of the validation process could reveal information about the resource to the attacker. For example the attacker could potentially generate a hash of an "obvious" response and see who can actually load it, or generate several integrity values and see which one works for a victim.

To prevent exactly this kind of abuse, the resource integrity validation is only performed when the same-origin policy (see my [CSRF post] for more info on SOP) is obeyed. This in essence means that integrity validation can only occur in the following two situations:

* the resource is served on the same origin as the page that includes it, or
* the resource is served on a different origin and [CORS] has been configured on the remote server (using the *Access-Control-Allow-Origin* header)

Note that inline resources are never validated for their integrity.

To ensure that cross-origin resources are validated correctly for your site, make sure that the relevant `script` and `link` tags have the [crossorigin] attribute with the appropriate value defined.

One thing to note here is that while the same origin requirement is very handy from security point of view, it also adds an additional constraint on CDN providers. Application developers may also not be able to leverage the integrity attribute until their third party providers set up CORS on their servers (the alternative is to self-host the resource, which is not always viable).

### Handling integrity validation errors

The integrity validation should succeed in the vast majority of the cases, however there are certain scenarios when it can go sideways, a non-exhaustive list would include:

* the CDN is not available
* there is a network error
* the resource is compromised
* there is an [optimizing proxy] changing HTTP responses

In any of these scenarios it is helpful to have a fallback mechanism to still load the resources your web application requires. As I was locally testing this out I've found that this part of the SRI support is not really that solid, and there are lots of different concerns that need to be taken into account. In the following subsections you can find the different issues I have faced as I was trying to implement fallback.

#### There is no global event for validation failure

Neither Chrome nor Firefox appears to call `window.onerror` or `error` event listeners when a script integrity check fails, which according to my understanding means that one has to register error listeners for every single &lt;script&gt; and &lt;link&gt; elements in the application. Registering these listeners can't really happen when the DOM is ready because the resource may be loaded by that time, so it looks like that the event handlers have to be registered as and when the resources are added to the DOM. To do this one can use [MutationObserver]s, for example:

```javascript
var observer = window.MutationObserver || window.WebKitMutationObserver;
if (observer) {
  new MutationObserver(function(mutations) {
    mutations.forEach(function(mutation) {
      mutation.addedNodes.forEach(processNode);
    });
  }).observe(document, { childList: true, subtree: true });
}
```

#### Loading the fallback script requires exceptions

Since the error handling script has to be defined before using any integrity protected scripts, one has to make sure that the fallback script can be loaded securely. Here you have a couple of options:

* Load it as an external script with integrity protection, but then you can't have a fallback code for that script.
* Load it as an external script without integrity protection, but then you can't use `require-sri-for script` directive in your CSP.
* Inline your fallback code into the HTML source, and use the integrity hash or nonce in CSP `script-src` to ensure that the script wasn't modified (and to prevent using `unsafe-inline`).

#### Beware of infinite loops

Given that as part of the error handling a new `script` element has to be introduced to the DOM, make sure that the MutationObserver's event handler can differentiate the original script nodes from the nodes that were added as fallback. An example snippet for this would look something like:

```javascript
if (node.tagName && node.tagName.toLowerCase() === 'script' && !node.getAttribute('data-retry')) {
  if (!node.onerror) {
    node.onerror = function(error) {
      var newNode = node.cloneNode();
      newNode.setAttribute("src", node.getAttribute('data-fallback-url'));
      newNode.setAttribute('data-retry', '1');
      newNode.onerror = function(error) { console.log('This failed too'); }
      node.parentNode.replaceChild(newNode, node);
    }
  }
}
```

#### The nodes have to be manually cloned

In the previous section the `node.cloneNode()` snippet isn't actually working... [It turns out][script clone] that cloned script elements are not loaded when they are added to the DOM. This means that the relevant `link` and `script` elements need to be cloned manually, a snippet like the following should be able to do the job:

```javascript
var attrs = node.getAttributeNames();
for (var i = 0; i < attrs.length; i++) {
  newNode.setAttribute(attrs[i], node.getAttribute(attrs[i]));
}
```

The above snippet will also copy over the `crossorigin` attribute if it was present, so if the fallback URL pointed to the current domain, you may wish to remove that attribute from the new node.

#### The new script tags aren't executed in the original order

When a JS file fails to load, there is a very good chance that its replacement `script` tag will only get executed after other scripts have already been executed. This can result in unrecognized reference issues and alike. This is by far the most difficult issue to address, and after a bit of trial and error and lots of reading, I think my takeaway is that the solution to this problem largely depends on the application in question: how it loads JavaScript code, and what browsers you wish to support...

Unfortunately there are lots of different ways out there to load JavaScript, here's how a few of them can be handled for fallback:

* every script is `async`: This is the best case of all, the SRI fallback implementation doesn't really have any requirements since async scripts never guarantee execution border.
* every script has `defer`: In this case the fallback script has to make sure that the replacement script tags are placed at the exact same location as the original one. [Some articles out there][script loading] also suggest that the async flag needs to be explicitly set to false, but I haven't seen this personally.
* every script is loaded synchronously: Worst possible scenario, and I'm not quite sure just yet that there is a good solution for this (other than using some dependency management library)
* dynamically loaded scripts: In this case fallback should be implemented as part of the dynamic loading mechanism.

As part of my investigations I have found the following resources to be very helpful, should you wish to implement fallback mechanism on your own:

* [Difference between async and defer attributes][async vs defer]
* [Script loading behavior in reality][script loading]
* [How to defer inline script][defer inline script]

I also had to realize that there are some restrictions on how one should load the SRI fallback script:

* It cannot be `async`, because you want to be sure that it's loaded before any other scripts (otherwise the error handler wouldn't be registered/called).
* It cannot be `defer`red, because by the time the MutationObserver would be registered, it would have missed most of the resources that are part of the DOM already.

#### An example fallback implementation

After all this struggle I came up with the following *example* code for SRI fallback:

```javascript
var observer = window.MutationObserver || window.WebKitMutationObserver;
if (observer) {
  new MutationObserver(function(mutations) {
    mutations.forEach(function(mutation) {
      mutation.addedNodes.forEach(processNode);
    });
  }).observe(document, { childList: true, subtree: true });
}

var processNode = function(node) {
  var tagName = node.tagName ? node.tagName.toLowerCase() : '';
  if (tagName === 'script' || tagName === 'link') {
    if (!node.onerror) {
      node.onerror = function(error) {
        var newNode = document.createElement(tagName);
        var attrs = node.getAttributeNames();
        for (var i = 0; i < attrs.length; i++) {
          newNode.setAttribute(attrs[i], node.getAttribute(attrs[i]));
        }
        if (tagName === 'script') {
          newNode.setAttribute('src', node.getAttribute('data-fallback-url'));
        } else {
          newNode.setAttribute('href', node.getAttribute('data-fallback-url'));
        }

        newNode.removeAttribute('data-fallback-url');
        newNode.onerror = function(error) { console.log('This failed too'); }
        node.parentNode.replaceChild(newNode, node);
      }
    }
  }
}
```

This code seemed to work well for my simple demo page, your mileage may vary. 😉

### Using SRI in combination with CSP

[Content-Security-Policy][CSP] headers are really powerful allies when it comes to protecting websites from code injection attacks. Whilst there are lots of different directives defined in CSP, today I'm going to only look at one of them specifically: `require-sri-for`.

The `require-sri-for` directive essentially allows a site to say that in order to load every `style` and/or `script` resources for the page, a valid integrity attribute has to be present. Whilst the purpose of SRI is to mainly protect sites from rogue CDNs and third parties, require-sri-for also demands that you specify the integrity attribute for same-origin resources as well. This can result in an additional burden for the applications, and it will almost certainly require some sort of a build system to ensure that the integrity values are always up-to-date.

One unfortunate shortcoming of the existing standards is that they don't always allow the specification of the integrity attribute. Imagine that you have an integrity protected CSS file, which has an [@import] directive loading additional files. Checking the integrity of such resources is currently not possible (since the `@import` directive does not support the integrity argument), and hence those transitive dependencies will fail to load.

There is a similar situation with the Worker API's [importScripts] function. The Worker API allows one to execute JavaScript code in background threads  separately from page rendering processing, but again the `importScripts` function has no means to define the integrity hash for the imported resource.

My understanding is that due to the above limitations, browsers currently do not enable require-sri-for support by default. To actually leverage require-sri-for, one can easily enable them for their own browsers:

* On Chrome visit *chrome://flags* and enable `#enable-experimental-web-platform-features`
* On Firefox open up *about:config* and enable `security.csp.experimentalEnabled` property

Since the above settings are still *experimental*, only enable them if you know what you are doing.

#### CSP reporting

When using `require-sri-for` in combination with CSP reporting, it is possible to get reports of violations of the rule. Violation however is limited to not having the integrity attribute for the script/CSS file in question, reports are not sent when the integrity value is actually incorrect. To remedy this I've written a simple report JS code based on the [report-uri-js] project to send reports of integrity violations:

```javascript
var observer = window.MutationObserver || window.WebKitMutationObserver;
if (observer) {
  new MutationObserver(function(mutations) {
    mutations.forEach(function(mutation) {
      mutation.addedNodes.forEach(processNode);
    });
  }).observe(document, { childList: true, subtree: true });
}

var processNode = function(node) {
  var tagName = node.tagName ? node.tagName.toLowerCase() : '';
  if (tagName === 'script' || tagName === 'link') {
    if (!node.onerror) {
      node.onerror = function(error) {
        var json = {
          "csp-report": {
            "document-uri": window.top.location.href,
            "referrer": "",
            "blocked-uri": node.hasAttribute('src') ? node.getAttribute('src') : node.getAttribute('href'),
            "violated-directive": "invalid-integrity",
            "original-policy": "require-sri-for script"
          }
        };
        var xhr = new XMLHttpRequest();
        xhr.open('POST', 'https://mydomain.report-uri.com/r/d/csp/enforce', true);
        xhr.setRequestHeader('content-type', 'application/csp-report');
        xhr.send(JSON.stringify(json));
      }
    }
  }
}
```

The above code snippet can be very useful to see if a fallback solution is actually necessary for your site.

#### Tricks to get around require-sri-for

When it comes to inline resources, `require-sri-for` doesn't really offer any sort of protection, there is no integrity validation performed for them. One way to get around the require-sri-for directive is to inline the resources, for example with this code:

```javascript
$.getScript('https://example.com/cool.js');
```

jQuery's `getScript` functionality works differently for same-origin and cross-origin requests though: for same-origin scripts, the scripts are inlined by creating a new &lt;script&gt; element with the script contents as the "body" of the element, whilst cross-origin scripts are just loaded as regular script nodes with the `src` attribute set to the provided URL.

The first problem with the cross-origin approach is that it [doesn't currently support integrity hashes][jquery bug], hence it doesn't work with require-sri-for. This can be worked around by making an AJAX GET request for the JS contents and inline it manually, but this is probably not the brightest and safest idea of all (and the integrity still wouldn't be validated).

The second problem is that loading scripts with `getScript` requires a bit more permissive `connect-src` and `script-src` directives in CSP, connect-src to allow the AJAX GET call for the script, and script-src to allow the inline script or the cross-origin script reference.

#### Issues with third parties

Unfortunately not all third party services want to support integrity verifications, one such example is [Google Fonts]. In their case the resources returned at their endpoints can be different depending on what type of browser was used to make the request and versioning the resources appeared to be too much of an overhead for them. Other than self-hosting these resources, there isn't much else that can be done unfortunately.

### TL;DR I want SRI now, what do I need to do?

To start to use SRI, just follow these simple steps:

* Go through your application's external resources and check if they are hosted on CORS enabled servers.
* Use online tools like [srihash.org] to generate the integrity hashes. If you have trust issues, you can also just execute the following command to get the hash:

```javascript
openssl dgst -sha512 -binary cool.js | openssl base64 -A
```


* Change the relevant `script` and `link` tags to have the `integrity` attribute set and make sure that the `crossorigin` attribute is also set (preferably with `anonymous` value).
* If you want to go for the extra mile and have local resources integrity verified as well, then you may wish to set the `Cache-Control: no-transform` header for those resources to ensure that optimizing proxies aren't causing any problems.
* Implementing fallback logic for your resource loading can result in a lot of work, and it only really covers the <1% likely scenario when a validation fails, so don't be too worried about it - just make sure you have reporting set up.

Whilst `require-sri-for` is a nice concept on paper, I don't believe that it has worth all the trouble just yet, especially since browsers most of the time ignore this directive at the moment.

[motherboard]: https://motherboard.vice.com/en_us/article/bj5m4v/cryptocurrency-mining-coinhive-browsealoud-hack-thousands-of-sites-catastrophe
[Scott Helme]: https://scotthelme.co.uk/protect-site-from-cryptojacking-csp-sri/
[Troy Hunt]: https://www.troyhunt.com/the-javascript-supply-chain-paradox-sri-csp-and-trust-in-third-party-libraries/
[CSP]: https://content-security-policy.com/
[Subresource Integrity]: https://www.w3.org/TR/SRI/
[optimizing proxy]: https://www.w3.org/TR/SRI/#proxies
[script clone]: https://stackoverflow.com/a/28771829
[CORS]: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
[CSRF post]: https://aldaris.github.io/dev/security/2017/10/11/cross-site-request-forgery.html
[crossorigin]: https://developer.mozilla.org/en-US/docs/Web/HTML/CORS_settings_attributes
[MutationObserver]: https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver
[async vs defer]: https://www.growingwiththeweb.com/2014/02/async-vs-defer-attributes.html
[script loading]: https://www.html5rocks.com/en/tutorials/speed/script-loading/
[defer inline script]: https://stackoverflow.com/a/41395202
[source list]: https://content-security-policy.com/#source_list
[@import]: https://developer.mozilla.org/en-US/docs/Web/CSS/@import
[importScripts]: https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/importScripts
[report-uri-js]: https://github.com/report-uri/report-uri-js/
[jquery bug]: https://github.com/jquery/jquery/issues/3028
[Google Fonts]: https://github.com/google/fonts/issues/473
[srihash.org]: https://www.srihash.org/
