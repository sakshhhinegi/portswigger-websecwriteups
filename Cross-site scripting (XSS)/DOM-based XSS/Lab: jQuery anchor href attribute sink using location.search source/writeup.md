## About the Lab

 **Difficulty:** Apprentice

 **Category:** Cross-site Scripting (XSS) - DOM-based

 **Lab URL:** [Lab: DOM XSS in jQuery anchor `href` attribute sink using `location.search` source](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-href-attribute-sink)
## Vulnerability Summary

The app contains a DOM-based XSS vulnerability in the submit feedback page. The `jQuery` function extracts a parameter from the user-controllable URL through `window.location.search` and directly assigns it to the `href` attribute of a HTML element. Because of this, we can supply our `JavaScript` payload through the URL to exploit XSS. The application uses jQuery to select the "back" link and modifies its `href` attribute using data obtained from `location.search`. Since the value is controlled through the URL, I was able to manipulate the `href` attribute and make the link execute JavaScript when clicked.
## Reconnaissance

Intercept the request to view the submit feedback page (`GET /feedback?returnPath=/`). The response body reads:
```html
 <script>
    $(function() {
	    $('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
    });
</script>
```
The `jQuery`'s `attr()` function, which can change the attributes of DOM elements, is used to change the `href` attribute of a HTML element using unsanitized data from `location.search`. As this data is user-controllable, we can use this as our source to inject a `JavaScript` payload.

An HTML link can use a `javascript:` URL as its destination. Therefore, if the application allows an attacker-controlled value to become the `href`, the link can potentially be converted from a normal navigation link into a JavaScript execution point.
## Exploitation Steps

1. Intercept the request to view the submit feedback page: `GET /feedback?returnPath=/`
2. On the Submit feedback page, change the query parameter returnPath to / followed by a random alphanumeric string `"abc"`.
3. Inspect the element, and observe that your random string `"abc"` has been placed inside an a `href` attribute. 
4. Change the value of the `returnPath` parameter to `javascript:alert(document.cookie)`, and send the request `GET /feedback?returnPath=javascript:alert(document.cookie)`
5. Observe that the lab is marked as solved on the browser.
## Payload Used

`javascript:alert(document.cookie)`

As the `new URLSearchParams(...).get('returnPath')` parses the query string and extracts the specific string value of the `returnPath` parameter, injecting our payload to this parameter makes the `jQuery` selector sets its `href` attribute to our payload. When a victim clicks the `#backlink` element, the payload triggers.
## Root Cause

The root cause is the assignment of attacker-controlled data to an anchor's `href` attribute without validating or restricting the value to safe URL schemes. It allows `javascript:` instead of strictly enforcing `http(s):`. Because the application allows the attacker to influence the complete `href` value, the `javascript:` scheme can be supplied as the link destination, turning the legitimate navigation link into a JavaScript execution point.
## Impact

An attacker could potentially craft a malicious URL that modifies the "back" link and causes JavaScript to execute when the victim interacts with it. Depending on the application's functionality and security controls, DOM-based XSS could allow an attacker to:
- Execute JavaScript in the context of the vulnerable application.
- Access data exposed to JavaScript.
- Perform actions within the victim's session.
- Manipulate page content.
- Conduct phishing or other client-side attacks.

In this lab, the impact was demonstrated by reading document.cookie.

## Remediation

The application should not directly assign untrusted URL parameters to security-sensitive attributes such as `href`. Recommended defenses include:

- Validate URL parameters before using them.
- Allowlist expected URL schemes such as `https:` or relative URLs.
- Explicitly reject dangerous schemes such as `javascript:`.
- Avoid using attacker-controlled values as complete URLs.
- Use safe DOM APIs and framework mechanisms where possible.
- Treat all URL-derived data as untrusted input.
- Consider Content Security Policy (CSP) as an additional defense layer.
