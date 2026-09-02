## About the Lab

 **Difficulty:** Practitioner

 **Category:** Cross-site Scripting (XSS) - DOM-based

 **Lab URL:** [Lab: DOM XSS in `document.write` sink using source `location.search` inside a select element](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink-inside-select-element)

## Vulnerability Summary

The vulnerability occurs because user-controlled data from `location.search` reaches the `document.write()` sink without being safely handled. This lab contains a DOM-based Cross-Site Scripting (XSS) vulnerability in the stock checker functionality. The application takes the `storeId` parameter from the URL using `location.search` and passes it to `document.write()`. Since the value is controlled through the URL and is written directly into the page, I was able to manipulate the generated HTML and break out of the existing `<select>` element to execute JavaScript. The objective was to exploit this behavior and trigger the `alert()` function.

Unlike reflected XSS, the malicious input is processed by client-side JavaScript rather than being reflected into the response by the server. 
## Reconnaissance

Send a request to view a product (`GET /product?productId=1`) results in a response that reads:
```html
<script>
    var stores = ["London", "Paris", "Milan"];
    var store = (new URLSearchParams(window.location.search)).get('storeId');
    document.write('<select name="storeId">');
    if (store) {
        document.write('<option selected>' + store + '</option>');
    }
    for (var i = 0; i < stores.length; i++) {
        if (stores[i] === store) {
            continue;
        }
        document.write('<option>' + stores[i] + '</option>');
    }
    document.write('</select>');
</script>
```
The `storeId` parameter is extracted from the `location.search` source. The script then uses `document.write` to create a new option in the `select` element for the stock checker functionality. There are no checks to see if this option existed in the `stores` array. 

Trying to create our own `storeId` parameter with a request `GET /product?productId=1&storeId=test`, we get a `200 OK` HTTP Response. Opening this response in the browser, we see that `test` is listed as one of the options of the product.

To exploit this vulnerability, we will need to break out of the `option` context and the `select` element first. We see that the script manually does this by using `document.write` to introduce closing tag `</select>` - we can introduce our own closing tag too, since nothing is HTML-encoded.
## Exploitation Steps

1. Opened the stock checker functionality. Observed the `storeId` parameter in the URL and traced the parameter through the client-side JavaScript.
2. Identified `location.search` as the source of the attacker-controlled data.
3. Identified `document.write()` as the dangerous sink and determined that the value was being written inside a `<select>` element.
4. Modify the request line to `GET product?productId=1&storeId=%22%3E%3C/option%3E%3C/select%3E%3Cimg%20src=1%20onerror=alert(1)%3E` and send the request. Lab should be marked as solved.
## Payload Used

`</option></select>img%20src=1%20onerror=alert(1)%3E` (URL encoded version of the payload: `</option></select><img src=1 onerror=alert(1)>`).
- `</option>` closes the existing `<option>` tag
 - `</select>` close the existing `select` element
 - `<img src=1 onerror=alert(1)>` is a XSS payload that triggers the `alert()` function when the element has focus.
## Root Cause

The page contains a JavaScript sink (`document.write`) that receives data from a taint source (`location.search`) without any sanitization or encoding. The application does not properly treat the storeId value as untrusted data before inserting it into the HTML DOM. As a result, an attacker can manipulate the HTML structure and introduce executable content. Because this happens entirely in the browser, standard server-side output encoding offers no protection — the vulnerability exists purely in the client-side code.
## Impact

An attacker could potentially craft a malicious URL that causes JavaScript to execute in the context of the vulnerable application when a victim visits it. Depending on the application's functionality and security controls, DOM-based XSS can potentially allow an attacker to:
- Manipulate page content.
- Perform actions within the victim's session.
- Access sensitive information exposed to JavaScript.
- Phish users through modified page content.
- Abuse application functionality available to the victim.
## Remediation

-  Replace `document.write` with safe DOM APIs that do not parse HTML. Use `textContent` or `createElement` + `setAttribute` instead:
```javascript
let selectElement = document.querySelector('select[name="storeId"]');
let newOption = document.createElement('option');
newOption.textContent = store; // textContent safely encodes HTML entities
selectElement.appendChild(newOption);
```
- Apply a strict Content Security Policy to block inline event handlers (`script-src 'self'`).
- Avoid document.write() for handling user-controlled data.
- Treat all URL parameters as untrusted input.
- Use safe DOM APIs such as textContent where HTML is not required.
- Apply context-appropriate output encoding when inserting data into HTML.
- Validate storeId against the expected format and allowed values.
- Consider Content Security Policy (CSP) as an additional defense layer.
