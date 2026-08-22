## About the Lab

 **Difficulty:** Practitioner

 **Category:** Access Control Vulnerabilities

 **Lab URL:** [Lab: URL-based access control can be circumvented](https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented)

## Vulnerability Summary

The application has an unauthenticated admin panel at /admin. However, a front-end security layer is configured to block direct external requests to this path.

The backend supports the X-Original-URL HTTP header, which can be used to make the backend process a different URL from the one initially requested.
This creates a mismatch between the URL checked by the front-end security layer and the URL processed by the backend, allowing the admin panel to be accessed without proper authorization.
## Reconnaissance

I first tried accessing /admin directly.

The request was blocked by the front-end security layer, confirming that external access to the admin path was restricted.

I then considered how the front-end and backend were handling the request and found that the backend framework supported the X-Original-URL header.

The important observation was that the application had two layers:

Client

   ↓

Frontend security layer

   ↓

Backend application

The front-end was responsible for blocking /admin, while the backend was responsible for serving the actual application functionality.

## Testing & Reasoning

Since the backend supported X-Original-URL, I tested whether I could request another URL through the header while keeping the visible request path different.

The idea was to see whether the front-end would check the normal URL while the backend would use the value supplied in X-Original-URL.
## Exploitation Steps

1. Navigate to `url/admin` and intercept this request.
2. In the intercepted request, modify the request headers as follows:
```
GET / HTTP/1.1
X-Original-URL: /admin/
...
```
and send the request. You should get a `200 OK`, and access to the admin panel. We see that the path to delete user `carlos` is `/admin/delete?username=carlos`.

3. Modify the intercepted request headers as follows:
```
GET /?username=carlos HTTP/1.1
X-Original-URL: /admin/delete
...
```
and send the request. You should get a `302 Found` and `Location: /admin` HTTP Response. Go on the website, and observe that lab is solved.
## Payload Used

To get the path for deleting `carlos`:
```
GET / HTTP/1.1
X-Original-URL: /admin
```
To delete `carlos`:
```
GET /?username=carlos HTTP/1.1
X-Original-URL: /admin/delete
```
The `X-Original-URL` is typically used to tell the backend app which URL the client originally requested before an intermediary (frontend server or proxy) modified it. In this lab, it is used to override the default request path.
## Root Cause

The access control was implemented at the front-end layer without ensuring that the backend enforced the same authorization rules.Because the backend trusted the X-Original-URL header, an attacker could manipulate the request so that the frontend and backend interpreted the requested resource differently.
The main issue is therefore a discrepancy between security controls at different application layers.
## Impact

An attacker could bypass the front-end restriction and access functionality that was intended to be protected.
If sensitive administrative functionality is exposed this way, an unauthenticated attacker could potentially perform privileged actions.
## Remediation

Access control should not depend solely on a front-end filtering layer.

Enforce authorization directly in the backend application.

Ensure that the backend does not blindly trust headers such as X-Original-URL when determining the requested resource.

Keep URL interpretation consistent between the frontend and backend.

Restrict or remove support for headers that allow clients to override the requested URL when they are not required.

Verify the user's permissions before serving sensitive functionality.
