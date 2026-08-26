## About the Lab

 **Difficulty:** Apprentice

 **Category:** Cross-site Scripting (XSS) - Stored

 **Lab URL:** [Lab: Stored XSS into HTML context with nothing encoded](https://portswigger.net/web-security/cross-site-scripting/stored/lab-html-context-nothing-encoded)
## Vulnerability Summary

The comment functionality stores user input and later renders it directly into the HTML page without encoding it. Because the application does not properly handle the submitted HTML, a `<script>` tag can be stored and subsequently executed whenever the affected blog post is viewed, making this a stored (persistent) XSS vulnerability. The objective was to exploit a stored cross-site scripting vulnerability by submitting a comment that executes the `alert()` function when the blog post is viewed.
## Reconnaissance

1. Submitting a normal comment on a blog post shows the comment appearing on the page after submission. The comment body is reflected verbatim in the HTML source between `<p>` tags with no encoding applied.
2. Testing `<b>hello</b>` in the comment body shows the text rendered in bold in the browser, confirming that HTML tags in comments are not stripped or encoded before being inserted into the page.
## Exploitation Steps

1. Navigate to any blog post on the lab.
2. Submitted a test comment.
3. Confirmed that the comment was stored and displayed when the page was viewed.
4. Fill in the comment form with any values for Name, Email, and Website, and enter `<script>alert(1)</script>` as the comment body.
5. Submitted the XSS payload.
6. Navigate back to the blog post (or reload it). The stored payload executes, triggering an `alert(1)` dialog, confirming successful stored XSS.
## Payload Used

`<script>alert(1)</script>`
Unlike reflected XSS, the payload is stored server-side and injected into every page render of the blog post. When any user (including the attacker themselves, or a victim tricked into visiting the post) loads the page, the browser encounters the `<script>` block and executes it as JavaScript. The impact is broader than reflected XSS because no victim interaction with a crafted URL is required — simply visiting the page is enough.
## Root Cause

The root cause is the lack of appropriate output encoding when rendering stored user input. The application accepts the comment, stores it, and later inserts it into the HTML response without encoding HTML-special characters. Because the browser interprets the stored input as HTML, an attacker can persist executable JavaScript in the application.
## Impact

Stored XSS can affect multiple visitors, not just the attacker who submitted the payload. Anyone viewing the affected blog post could execute the attacker's JavaScript in their browser. Depending on the application's functionality and security controls, this could allow an attacker to perform actions as the victim, manipulate page content, or access sensitive browser-accessible information.
## Remediation

HTML-encode all stored user content on output, before inserting it into any HTML context:
````python
from markupsafe import escape

comment_body = escape(stored_comment)
html = f"<p>{comment_body}</p>"
````
Additionally, consider a Content Security Policy (CSP) with `script-src 'self'` as a defence-in-depth measure to block inline script execution even if encoding is ever missed.

The application should treat stored comments as untrusted data and apply appropriate output encoding whenever they are rendered.
1. HTML-encode user-generated comments before displaying them.
2. Use secure templating mechanisms that automatically encode output.
3. Avoid inserting untrusted content directly into HTML.
4. Apply context-appropriate XSS protections wherever user-generated content is rendered.
5. Use a properly configured Content Security Policy (CSP) as an additional defense layer.
