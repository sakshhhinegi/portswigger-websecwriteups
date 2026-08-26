## About the Lab

 **Difficulty:** Apprentice

 **Category:** Cross-site Scripting (XSS) - Reflected

 **Lab URL:** [Lab: Reflected XSS into HTML context with nothing encoded](https://portswigger.net/web-security/cross-site-scripting/reflected/lab-html-context-nothing-encoded)
## Vulnerability Summary

The search parameter is reflected directly into the HTML page without being encoded or properly sanitized. Because the input is placed into an HTML context, I was able to inject a <script> tag containing JavaScript. The browser interpreted the reflected input as HTML and executed the JavaScript. The objective was to execute JavaScript through a reflected cross-site scripting (XSS) vulnerability by calling the alert() function.
## Reconnaissance

 Entering a normal string like `hello` into the search bar produces this request: `GET /?search=hello`, and the string is reflected back in the response body inside a `<h1>` element: `<h1>1 search results for 'hello'</h1>`. The input was returned directly in the HTML response, indicating that the application was not properly encoding the supplied input before rendering it.
 Testing `<` in the search field shows the character rendered literally in the page source, not as `&lt;`. No HTML encoding is applied to user input.
## Exploitation Steps

1. Navigate to the lab's search functionality.
2. Observed that the value was reflected in the response.
3. Tested HTML markup to determine whether the input was being encoded.
4. Enter `<script>alert(1)</script>` in the search box and submit. The browser parsed the injected markup and executed the JavaScript.
5. The server reflects the payload back into the page and the browser executes it, triggering an `alert(1)` dialog, confirming successful XSS execution.
## Payload Used

`<script>alert(1)</script>`
The search term is reflected directly into the HTML body with no encoding, so any injected tag is treated as valid markup by the browser. The `<script>` tag tells the browser to execute its contents as JavaScript, causing `alert(1)` to run in the victim's origin.
## Request / Response Analysis

The important behavior was that the search input appeared directly in the HTML response. Conceptually, the application was doing: `<p>Search results for: [user input]</p>` Instead of safely encoding the input, the application allowed HTML supplied by the user to become part of the page structure.
As a result, the injected `<script>` element was interpreted by the browser and executed.

## Root Cause

The root cause is **insufficient output encoding**. The application takes user-controlled input from the `search` parameter and reflects it into an HTML response without encoding characters that have special meaning in HTML. Because the browser cannot distinguish the injected markup from legitimate application markup, attacker-controlled JavaScript can be executed in the context of the vulnerable page.
## Remediation

HTML-encode all user-supplied values before inserting them into the page. In most server-side frameworks this is the default behavior of the templating engine:
````python
from markupsafe import escape

search_term = escape(request.args.get("search", ""))
html = f"<h1>Search results for '{search_term}'</h1>"
````
`<` becomes `&lt;`, `>` becomes `&gt;`, so the injected tag is displayed as text rather than interpreted as markup.

The application should treat user input as untrusted and apply context-appropriate output encoding before inserting it into HTML.
- HTML-encode user-controlled data before rendering it.
- Use safe templating mechanisms that automatically encode output.
- Avoid inserting untrusted input directly into HTML.
- Validate input where appropriate, but do not rely on input validation as the primary XSS defense.
- Consider a properly configured Content Security Policy (CSP) as an additional defense layer.
