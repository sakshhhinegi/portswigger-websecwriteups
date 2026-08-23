## About the Lab

 **Difficulty:** Practitioner 

 **Category:** Access Control Vulnerabilities

 **Lab URL:** [Lab: Referer-based access control](https://portswigger.net/web-security/access-control/lab-referer-based-access-control)
## Vulnerability Summary

The application relies on the `Referer` header to determine whether a user is allowed to access an administrative function.
Because the Referer header is controlled by the client, it can exploit this by modifying the value of this header to perform vertical privilege escalation. This makes it an unreliable mechanism for enforcing authorization.
## Reconnaissance

Navigate to `url/login` and log in with credentials `administrator - admin`. Intercept the request to upgrade the role of user `carlos` (from `NORMAL` to `ADMIN`). We notice that the value of the `Referer` header is `url/admin`. Now log out, then log in to `wiener`'s account (`wiener - peter`). Try to perform the same upgrade request from `wiener`'s account - you will get a `401 Unauthorized` HTTP Response. Exact same request but with the header `Referer: https://<lab-id>.web-security-academy.net/admin` yields a `302 Found` HTTP Response with header `Location: /admin`. This suggests that the app only perform access control checks by inspecting the `Referer` header.
## Exploitation Steps

1. Logged into the administrator account to understand the promote-user functionality. 

2. Captured the promote-user request using Burp Suite.

3. Inspected the request, including the Referer header in the request: `Referer: https://<lab-id>.web-security-academy.net/admin`.

4. Tested changing the HTTP method and observed how the server responded.

5. Logged in as wiener and captured the user's session cookie.

6. Replayed the administrative request using the wiener session.

7. Change the header of the intercepted `GET` request to `GET /admin-roles?username=wiener&action=upgrade HTTP/1.1`

8. Supplied the required Referer value.

9. The server accepted the request and promoted wiener to administrator. You should get a `302 Found` HTTP Response with `Location: /admin`.
## Payload Used

`Referer: https://<lab-id>.web-security-academy.net/admin`
The app only checks the value of the `Referer` header for the role upgrade/downgrade functionality. Thus, as long as we have this header, we can upgrade/downgrade the role of any arbitrary user.
## Root Cause

The application uses the `Referer` header as part of its authorization mechanism.
However, the `Referer` header is supplied by the client and can be manipulated. It therefore cannot be trusted to determine whether a user has administrator privileges.
The application also failed to enforce authorization consistently when the request method was changed.
The core issue is the absence of a reliable server-side authorization check based on the authenticated user's actual privileges.
## Impact

A lower-privileged user could bypass the intended access control and perform an administrative action.
In this lab, this allowed the wiener account to be promoted to administrator without having administrator credentials.
In a real application, similar weaknesses could allow unauthorized users to perform sensitive administrative operations.
## Remediation

The application should never use the Referer header as an authorization mechanism.

- Perform authorization checks using the authenticated user's server-side session and assigned role.
- Verify administrator privileges before processing sensitive actions.
- Enforce authorization consistently across different HTTP methods.
- Do not rely on client-controlled headers to determine permissions.
- eturn an authorization error when a user without the required privileges attempts an administrative action.
