## About the Lab

**Difficulty:** Apprentice

**Category:** Cross-site Scripting (XSS) - DOM-based

**Lab URL:** [Lab: DOM XSS in jQuery selector sink using a hashchange event](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-selector-hash-change-event)
## Vulnerability Summary

The app contains a DOM-based XSS vulnerability in the homepage. It uses `jQuery`'s `$()` selector function to auto-scroll to a given post, whose title is passed via the `location.hash` property, in the event of `hashchange`. This means that we can deliver an exploit to force the victim's browser to load the page first, change the hash without user interaction (which fires `hashchange`), then trigger our XSS payload.

The application uses jQuery's `$()` selector function to locate a blog post based on its title. The title is obtained from the URL's `location.hash` property and processed when a `hashchange` event occurs. Because the hash value is user-controlled and is passed into a jQuery selector without proper handling, I was able to manipulate the selector and execute JavaScript.
## Reconnaissance

Intercept the request to view the homepage (`GET /`). The response body reads:
```html
<script src="/resources/js/jquery_1-8-2.js"></script>
<script src="/resources/js/jqueryMigrate_1-4-1.js"></script>
<script>
    $(window).on('hashchange', function() {
	    var post = $('section.blog-list h2:contains(' + decodeURIComponent(window.location.hash.slice(1)) + ')');
        if (post) post.get(0).scrollIntoView();
	});
</script>
```
- The event listener waits for URL's fragment identifier (begins with `#`) to change to trigger the `hashchange` event.
- When the `hashchange` event fires, `location.hash.slice(1)` extracts the user-controllable fragment payload, stripping away the `#` character.
- The URL decoded payload (after `decodeURIComponent`) is concatenated directly into a `jQuery` selector string and passed to the `$()` function. The app's intention is to find a `h2` header containing this value to automatically scroll into it.
- As the app is using `jQuery` version `1.8.2`, the `$()` function can be our sink, as `jQuery` will parse and instantiate whatever HTML tags we insert into it.
## Testing & Reasoning

The important part of this lab was understanding the interaction between the URL hash, the `hashchange` event, and the jQuery selector. The application expected the hash to contain the title of a legitimate post and used that value to construct a selector. Since the selector was influenced by attacker-controlled data, I investigated whether I could inject a selector that caused jQuery to interpret my input differently from the application's expected post title.

The exploit also needed to account for the `hashchange` event. Simply supplying a malicious hash was not sufficient for the intended exploit delivery. The victim first needs to load the page, after which the hash can be changed. Changing the fragment triggers the `hashchange` event, causing the vulnerable JavaScript to process the attacker-controlled value. This was an important part of the lab because it demonstrated that DOM XSS can depend not only on the vulnerable sink, but also on browser events that cause the vulnerable code to execute.
## Exploitation Steps

1. Go to the exploit server.
2. Examined how the page handled the URL fragment. Identified that the application used `location.hash` to determine the blog post to scroll to.
3. Inspected the client-side JavaScript and identified the `hashchange` event. Traced the hash value to jQuery's `$()` selector.
4. Determined that the selector was influenced by attacker-controlled input.
5. Tested how modifying the hash affected the selector and DOM behavior.
6. Store this payload `<iframe src = "https://LAB-ID.web-security-academy.net/#" onload="this.src+='<img src=1 onerror=print()>'"></iframe>` and deliver exploit to victim.
7. Observe that lab is marked as solved. 
## Payload Used

`<iframe src = "https://LAB-ID.web-security-academy.net/#" onload="this.src+='<img src=1 onerror=print()>'"></iframe>`
- An inline frame is used on the exploit server to embed the victim's session of the app.
- The `onload` event waits until the `iframe` has completely finished loading the target's page. That way the `hashchange` event can be fired, and the payload can be executed.
- `this.src` accesses the current URL of the `iframe`, and the `+=` operator appends the XSS payload (`<img src=1 onerror=print()>`) directly to the end of the existing URL (ends in `#`).
## Root Cause

The vulnerability stems from passing user-controllable input (`window.location.hash`) directly into the jQuery `$()` selector function without validation or sanitization. In jQuery versions prior to `1.9.1` (the app uses `1.8.2)`, the `$()` function evaluates strings containing HTML tags and dynamically instantiates DOM elements. This allows an attacker to inject arbitrary HTML, including elements with malicious event handlers, which execute upon instantiation.
## Impact

An attacker could potentially construct an exploit that causes JavaScript to execute in the context of the vulnerable application when a victim visits the malicious URL. Depending on the application's functionality and security controls, DOM-based XSS could allow an attacker to:

- Execute JavaScript in the victim's browser.
- Manipulate page content.
- Perform actions within the victim's session.
- Access data exposed to JavaScript.
- Modify the page to conduct phishing attacks.
## Remediation

Upgrading jQuery to 1.9.1+ is a best practice but insufficient for this specific implementation. Because `window.location.hash.slice(1)` strips the leading `#`, an attacker can supply a payload starting with `<`. This bypasses jQuery 1.9+'s security checks, forcing the framework to parse the string as HTML rather than a selector.
To fix this, stop passing unsanitized user input directly into the `$()` selector function.
* Select the target elements first (e.g., all headers), then use jQuery's `.filter()` method to match their text content against the user input. This treats the input strictly as data rather than executable code.
* Enforce a whitelist (e.g., alphanumeric characters and hyphens only) on the hash value before processing it. 
* If concatenating user input into a selector string is unavoidable, sanitize the input using `CSS.escape()` to neutralize special characters, preventing them from being interpreted as HTML tags or malicious pseudo-classes.
