## About the Lab

**Difficulty:** Practitioner

**Category:** Cross-site Scripting (XSS) - DOM-based

**Lab URL:** [Lab: Reflected DOM XSS](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-dom-xss-reflected)
## Vulnerability Summary

The app contains a reflected DOM-based XSS vulnerability in the search functionality. User supplied input in the search bar is echoed in the response, which then is processed by an insecure script, granting us the sink to execute `JavaScript`.

The app reflects the search input into a JSON response called `search-results`. A client-side JavaScript file then processes this response using `eval()`.

Although the application escapes quotation marks in the JSON response, it does not escape backslashes. This allows the JSON structure to be manipulated so that the reflected input can break out of its intended string context and introduce JavaScript code into the `eval()` call.
## Reconnaissance

- Entering a random string like `test` into the search bar produces this request: `GET /?search=test`. In the response, the script `searchResults.js` is used to process our input, calling the `search()` function with the parameter `search-results` as the path.
- The script `searchResults.js`, more specifically the invoked `search()` function can be accessed on the source of the search result page. It reads:
```js
function search(path) {
    var xhr = new XMLHttpRequest();
    xhr.onreadystatechange = function() {
        if (this.readyState == 4 && this.status == 200) {
            eval('var searchResultsObj = ' + this.responseText);
            displaySearchResults(searchResultsObj);
        }
    };
    xhr.open("GET", path + window.location.search);
    xhr.send();
```
As we can see, our input from the search function is processed using the `eval()` function, which can be used to execute `JavaScript` code. This is our sink.
- We need to find out how our data is actually reflected. Along with the response to the string `test`, we can also see that response of the `search()` function. It reads:
```json
{
    "results": [],
    "searchTerm": "test"
}
```
The server reflects user input from the search function into a `JSON` response.
-  Try submitting a double quote (`"`) results in the response:
```json
{
    "results": [],
    "searchTerm": "\""
}
```
The server escapes it with a backslash (`\`).
- Try submitting a backslash (`\`) reveals that the server does *not* escape backslashes:
```json
{
    "results": [],
    "searchTerm": "\"}
```
- We can also break out of the `searchResultsObj` itself with `\"}`:
```json
{
    "results": [],
    "searchTerm": "\\"
}
"}
```
## Testing & Reasoning

Since the reflected value was being incorporated into JSON, I first tested how special characters were handled. I found that quotation marks were escaped. This prevented a straightforward attempt to terminate the JSON string using a quote. However, the application did not escape the backslash character.

This was significant because backslashes are used as escape characters inside JSON and JavaScript strings. If the escaping mechanism itself can be manipulated, the structure of the string passed to `eval()` may be altered.


## Exploitation Steps

1. Opened the application's search functionality. Submitted a normal search term "test" and intercepted the search request using Burp Suite.
2. Observed that the search term was reflected in the `search-results` JSON response. Located the `searchResults.js` file through the Site Map.
3. Inspected how the JSON response was processed and identified the use of `eval()` on the response.
4. Tested different special characters to understand the application's encoding behavior. 
5. Confirmed that quotation marks were escaped.
6. Tested the handling of backslashes and found that they were not escaped.
7. Used the backslash to interfere with the application's quotation-mark escaping.
8. Constructed an injection/input `\"-alert(1)}//` and search. 
9. The `alert()` function should be triggered, and lab is marked as solved.
## Payload Used

`\"-alert(1)}//`

With this payload, when passed to `eval()`, the execution context becomes:
```js
eval('var searchResultsObj = {"searchTerm":"\\"};alert(1);//"}');
```
- `var searchResultsObj = {"searchTerm":"\\"}` is parsed as a complete, valid statement assigning an object to the variable.
- The semicolon `;` terminates that statement.
- `alert(1);` is parsed and executed as a standalone statement.
- `//` comments out the trailing `"}`, preventing a syntax error at the end of the `eval()` function.

The important part of the payload is the backslash. The application escapes quotation marks but does not properly escape backslashes. This allows the supplied backslash to interact with the application's escaping and modify how the resulting JavaScript is interpreted by `eval()`.
## Root Cause

The fundamental security issue is treating attacker-controlled data as executable JavaScript.
1. The application fails to properly serialize `JSON`. It escapes double quotes but does not do so with backslashes.
2. The application also performs incomplete escaping: quotation marks are escaped, but backslashes are not handled correctly.
3. The dangerous, insecure use of the `eval()` function to process user supplied data, allowing `JavaScript` code to be executed.
## Impact

An attacker could potentially construct a malicious search URL that causes JavaScript to execute when a victim visits the URL. Depending on the application's functionality and security controls, DOM-based XSS could allow an attacker to:

- Execute arbitrary JavaScript in the victim's browser.
- Perform actions within the victim's session.
- Access data exposed to JavaScript.
- Manipulate page content.
- Conduct phishing or other client-side attacks.
- Abuse functionality available to the victim.
## Remediation

The application should never use `eval()` to parse or process untrusted data. Recommended defenses include:

- Replace `eval()` with `JSON.parse()` when processing JSON.
- Treat server responses as data rather than executable JavaScript.
- Apply correct JSON encoding to user-controlled values.
- Ensure backslashes and other special characters are properly escaped according to the output context.
- Avoid dynamically constructing JavaScript code from user input.
- Validate and constrain search input where appropriate.
- Use Content Security Policy (CSP) as an additional defense layer.
