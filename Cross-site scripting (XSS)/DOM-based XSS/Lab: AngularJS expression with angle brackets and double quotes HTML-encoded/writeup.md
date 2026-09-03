## About the Lab

**Difficulty:** Practitioner

**Category:** Cross-site Scripting (XSS) - DOM-based

**Lab URL:** [Lab: DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-angularjs-expression)
## Vulnerability Summary

The app contains a DOM-based XSS vulnerability in a `AngularJS` expression within the search functionality. User-supplied input (in the search box) is reflected directly into an HTML governed by `AngularJS`'s `ng-app` directive without proper sanitization. By escaping  local `$scope` object context and traversing up the prototype chain, we can reach the native `Function` object to execute our `JavaScript` payload.

The application uses AngularJS to process content inside an element governed by the `ng-app` directive. The search input is reflected into an AngularJS expression, allowing JavaScript expressions inside `{{ }}` to be evaluated. Although angle brackets and double quotes are HTML-encoded, this does not prevent AngularJS from interpreting expressions inside its template. Therefore, instead of trying to inject HTML, I could use AngularJS expression syntax to execute JavaScript.
## Reconnaissance

- Entering a random string like `test` into the search bar produces this request: `GET /?search=test`. The response to this request reads:
```html
<html>
<!--LAB_HEAD_START-->

<head>
    <link href=/resources/labheader/css/academyLabHeader.css rel=stylesheet>
    <link href=/resources/css/labsBlog.css rel=stylesheet>
    <script type="text/javascript" src="/resources/js/angular_1-7-7.js"></script>
    <title>DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded</title>
</head>
<!--LAB_HEAD_END-->

<body ng-app>
    <script src="/resources/labheader/js/labHeader.js"></script>
    <!--LAB_HEADER_START--> 
    ...
</body>
```
Our input to the search box is enclosed inside `AngularJS`'s `ng-app` directive.
- Inputting `{{7*7}}` into the search bar, we get a response in the browser as **'49'**. 
The multiplication, when placed inside double curly braces, is executed by `AngularJS` and its result is reflected in the response. 
- However, inputting `{{alert(1)}}` into the search bar does not trigger the `alert()` function. This is probably because that this function is a property of the global `window` object and is not defined within the local `$scope`.
## Exploitation Steps

1. Opened the application's search functionality. Submitted a normal search term `test` and observed where it appeared in the page.
2. Inspected the page structure and identified the AngularJS `ng-app` directive. Determined that the reflected search input was inside an AngularJS-controlled context.
3. Considered conventional HTML XSS payloads but recognized that angle brackets and double quotes were encoded.
4. Tested AngularJS expression syntax using `{{ }}`. Confirmed that AngularJS evaluated the supplied expression.
5. Traversed the prototype chain to reach the native `Function` constructor. Used the `Function` object to execute JavaScript.
6. Input `{{$on.constructor('alert(1)')()}}` and search. 
7. The `alert(1)` function should be triggered, and lab is marked as solved.
## Payload Used

`{{$on.constructor('alert(1)')()}}`
- The payload is placed inside double curly braces to make it be parsed and executed by the `AngularJS` expression evaluator rather than the `JavaScript` engine.
- `$on` is a built-in AngularJS scope method; its `.constructor` is also `Function`, achieving the same result via a different entry point.
- Passing the string `'alert(1)'` dynamically generates a new function: `function() { alert(1); }` in the **global** execution context (because it was created via the `Function` constructor).
- The final sets of parentheses `()` invokes the `alert()` function. As it is executed in the global context, it has full access to the `window` object, allowing `alert(1)` to execute successfully.
## Root Cause

User input is reflected directly into an HTML structure governed by the `ng-app` directive without proper sanitization. Even if angle brackets (`<`, `>`) and quotes (`"`) are HTML-encoded, AngularJS still parses and evaluates the expression within the curly braces. By escaping the local `$scope` object, we can execute malicious `JavaScript`. 
## Impact

An attacker could potentially construct a malicious search URL that causes JavaScript to execute in the context of the vulnerable application. Depending on the application's functionality and security controls, DOM-based XSS could allow an attacker to:

- Execute arbitrary JavaScript in the victim's browser.
- Manipulate page content.
- Perform actions within the victim's session.
- Access data exposed to JavaScript.
- Modify the page to conduct phishing attacks.
- Abuse functionality available to the victim.
## Remediation

- **Avoid Reflection inside `ng-app`:** Never reflect unsanitized user input directly into a DOM node controlled by AngularJS.
- **Use `ng-non-bindable`:** If user input must be displayed within an Angular application, wrap the container element with the `ng-non-bindable` directive. This instructs Angular to ignore interpolations within that specific DOM node.
- **Keep Frameworks Updated:** Modern versions of Angular (Angular 2+) use strict Contextual Auto-Escaping and Ahead-of-Time (AOT) compilation, which severely mitigates client-side template injection (CSTI) vulnerabilities.
