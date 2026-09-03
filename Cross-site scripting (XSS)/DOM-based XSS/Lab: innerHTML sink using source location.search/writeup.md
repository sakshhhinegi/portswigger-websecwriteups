## About the Lab

 **Difficulty:** Apprentice

 **Category:** Cross-site Scripting (XSS) - DOM-based

 **Lab URL:** [Lab: DOM XSS in `innerHTML` sink using source `location.search`](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-innerhtml-sink)
## Vulnerability Summary

The vulnerability occurs because user-controlled data from `location.search` reaches an `innerHTML` sink without being safely handled. Since the value is controlled through the URL and is interpreted as HTML, I was able to inject a new HTML element with an event handler and execute JavaScript. The objective was to exploit this behavior and trigger the `alert()` function.
## Reconnaissance

I started by interacting with the search functionality and observing how the search parameter was handled. Entering a random string like `test` into the search bar produces this request: `GET /?search=test`. Inspecting the rendered page shows the search term being inserted into the DOM by client-side JavaScript, rather than being safely rendered as text by the server.
```html
<script>
	function doSearchQuery(query) {
        document.getElementById('searchMessage').innerHTML = query;
    }
    var query = (new URLSearchParams(window.location.search)).get('search');
    if (query) {
        doSearchQuery(query);
    }
</script>
```
The vulnerable script reads from the `location.search` source and writes the search term into a `div` using the `innerHTML` sink. This is dangerous because the browser interprets the assigned value as markup.
## Exploitation Steps

1. Search for a random value, e.g. `test`. Intercept this request.
2. Observed the input was being assigned to `innerHTML`, I first considered whether I could inject an HTML element that would execute JavaScript. However, `innerHTML` does not execute injected `<script>` elements in modern browsers. Similarly, an `svg` element with an `onload` handler would not provide a working solution in this context.
3. Therefore, I needed to use another HTML element together with an event handler that would execute when the element encountered an error or loaded. An `<img>` element with an `onerror` event handler was suitable for this purpose. Replace the `search` parameter's value with the value below and send the request.
```
<img src=1 onerror=alert(1)>
```
When you go onto the browser, lab should be marked as solved. You can open the response to the request onto the browser to trigger the payload, confirming XSS.
## Payload Used

The payload is a partially URL-encoded version of:
```html
<img src=1 onerror=alert(1)>
```
- The invalid `src` causes the browser to generate an image-loading error, which triggers the `onerror` event handler and executes `alert(1)`.
## Root Cause

The page contains a client-side taint flow from `location.search` to `innerHTML`. User-controlled input from the URL is inserted into the DOM as executable markup without sanitization or encoding. Because this processing happens in the browser, server-side reflection checks alone are not enough; the vulnerable JavaScript sink is what turns the URL parameter into active DOM nodes and event handlers.
## Impact

An attacker could potentially craft a malicious URL that causes JavaScript to execute when a victim visits the vulnerable page. Depending on the application's functionality and security controls, DOM-based XSS could allow an attacker to:

- Manipulate page content.
- Perform actions within the victim's session.
- Access sensitive information exposed to JavaScript.
- Modify the page to conduct phishing attacks.
- Abuse functionality available to the victim.
## Remediation

- The application should avoid assigning untrusted URL parameters directly to `innerHTML`. Replace `innerHTML` with a safe text sink such as `textContent` when rendering search terms:
```javascript
document.getElementById("searchMessage").textContent = search;
```
- Apply a strict Content Security Policy via HTTP response headers. Ensure the `script-src` directive omits `'unsafe-inline'` so inline event handlers such as `onfocus` cannot execute even if HTML injection occurs.
- Avoid inserting untrusted data into HTML.
- Apply context-appropriate output encoding when HTML insertion is actually required.
- Validate URL parameters against the expected format.
- Avoid allowing user-controlled input to define HTML event handlers.
