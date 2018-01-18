---
layout: post
title:  "Credit card number validation gone wrong"
date:   2018-01-14 01:00:00 +0000
categories: dev js
---
When done right, online shopping can be a very nice and simple experience, but there are sadly also times when it is overly complicated and the user experience is just terrible. My experience on argos.co.uk not too long ago was the latter, mainly because I couldn't enter my credit card number at checkout. In the followings I will try to figure out why this was the case and try to offer a better implementation for them.

**NB: I'm not a JavaScript developer, so please take this post with a pinch of salt. Feel free to point out any mistakes made in the comment section, and I'll endeavour to rectify them.**

### The use-case

On the credit card form the user should be able to enter only numbers, no other characters shall be allowed on the form. Sounds like an easy enough job for JavaScript, right?

### The faulty code

When looking at the page source for the Argos payment page the following code snippet could be found in there:

```javascript
$("#cardnumber").on('keydown', function(event) {
  allowOnlyValidChars(event);
});
$("#securityCode").on('keydown', function(event) {
  allowOnlyValidChars(event);
});
function allowOnlyValidChars(event){
  var allowedCharKeys = [48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 8, 9, 35, 36, 37, 39, 46];
  if (event.shiftKey) {
    if (event.which != 9) {
      event.preventDefault();
      event.stopPropagation();
    }
  } else if ($.inArray(event.which, allowedCharKeys) == -1) {
    event.preventDefault();
    event.stopPropagation();
  }
}
```

So the gist of the code appears to be simple:

* if the Shift button is pressed and the TAB button isn't pressed, or
* if the Shift wasn't pressed and the list of allowed keys does not contain the **keycode** of the **event**, stop the event processing.

When the default event behavior is prevented, the input field is simply not updated with the character corresponding to the event.

### The issue

The problem with the above code snippet is that when an end-user tries to enter their credit card number on a foreign (non-English) keyboard layout, they are not allowed to enter the character **0** if the location of the **0** character is different on their layout. If the keyboard layout is changed to English, suddenly they are allowed to enter all the numbers...

### The root cause

There are three kinds of key related events in JavaScript: **keydown**, **keyup**, **keypress**. With a little bit of looking around one can find out a few things about them:

* In case of the **keydown** and **keyup** events the event handler will receive notifications about the [physical keys][not character codes] being pressed on the keyboard, whilst the **keypress** event [notifies about an actual character being entered][keydown keypress difference].
* The **keyCode** and the **which** fields in the **event** object are [deprecated][keycode deprecated], and instead people should be using the **key** field.

### The fix

To fix this annoying bug, one would just need to change the event registrations to use **keypress** instead of **keydown**, because the **keyCode** and **which** fields for that event type reports the character code and not the key code... Another fix would be to just simply prefer the **key** field in the **event** object [when it's available][caniuse].

An example fix would look like:

```javascript
$("#foo").on("keypress", function(event) {
  var charCode;
  if (event.key) {
    charCode = event.key.charCodeAt(0);
  } else {
    charCode = event.which;
  }
  if (charCode < 48 || charCode > 57) {
    event.preventDefault();
    event.stopPropagation();
  }
});
```

**UPDATE: A colleague of mine has [pointed out][html5] that it is also possible to use the HTML5 number input type to achieve the same goal, A working example for that would look something like this:**

```html
<input type="number" name="creditcard" />
```

```css
input[type='number']::-webkit-outer-spin-button,
input[type='number']::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
}
```

[keydown keypress difference]: https://stackoverflow.com/a/3396790
[not character codes]: https://stackoverflow.com/a/9350415
[keycode deprecated]: https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent
[caniuse]: https://caniuse.com/#feat=keyboardevent-key
[html5]: https://twitter.com/misterpotes/status/952811059282350080
