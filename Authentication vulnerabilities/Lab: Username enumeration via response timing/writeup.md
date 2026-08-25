## About the Lab

 **Difficulty:** Practitioner 

 **Category:** Authentication

 **Lab URL:** [Lab: Username enumeration via response timing](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing)

## Vulnerability Summary

The vulnerability is **CWE-208: Observable Timing Discrepancy**. The application is vulnerable to username enumeration through response timing. The application's login functionality behaves differently depending on whether the submitted username is valid. The difference can be identified through response timing. When an invalid username is submitted, the server responds relatively quickly. With a valid username and an incorrect password, the server takes noticeably longer to respond. This timing difference can be measured and used to identify valid usernames. The application also has IP-based brute-force protection, which adds another obstacle during password guessing. Though brute force rate limiting defense mechanism is implemented through IP blocking, using a bunch of random IP addresses with the HTTP header `X-Forwarded-For` easily bypasses this.
## Reconnaissance

I observed that requests using random/invalid usernames generally received a response faster than requests using a valid username with an incorrect password. This indicated that the application was processing the two cases differently and that response timing could potentially be used for username enumeration. The timing difference became more noticeable when using an excessively long password.
After a few attempts of credentials enumeration, the app rate limits us `You have made too many incorrect login attempts. Please try again in 30 minute(s).` However, adding the `X-Forwarded-For` HTTP header using a random IP address bypasses this mechanism. We can still bruteforce the username, then the password.
## Exploitation Steps

1. Go to `url/login`, and type in random username and password values to send a login request. Intercept this request with Burp Suite proxy and send it to Burp Suite **Intruder**.
2. First, The application applied brute-force protection based on the client's IP address.
I added: `X-Forwarded-For: <IP>`
and changed the IP value between requests. Set the first payload position to be the value of the `X-Forwarded-For` header from 1-100. Set the second payload position to be the value of the `username` field. On the **Payload configuration** settings, paste the **candidate usernames** list provided by the lab's description. Set the value of the `password` field to be exceptionally long. After doing all these things, perform a **Pitchfork Attack**. 

4. Sort the **Response received** column in descending order. You'll see exactly 2 responses that have a considerable larger **Response received** value, 1 of them is our username `wiener`. The other one is the valid username we're looking for.

5. The job now is easy. Change the value of the `username` field to the the valid username you just got, keep this fixed. Now we need to find the password for this username. Set payload position to be the value of the `password` field, paste the list of **candidate passwords** (also provided by the lab) onto the **Payload configuration** settings, then start a **Pitchfork Attack** again (DO keep the payload position and list of IP addresses we used earlier). After the attack's done, sort the responses by their length in **ascending** order. You should see that there's exactly 1 response that is significantly shorter in length from the others. When you check the details of this response, you should see that the status code is `302 Found`, indicating a redirection. Using the obtained username and this password to log in, you should success. Lab is solved!
## Payload Used

[Candidate username list](https://portswigger.net/web-security/authentication/auth-lab-usernames)

[Candidate password list](https://portswigger.net/web-security/authentication/auth-lab-passwords)

These lists are provided by PortSwigger.
## Root Cause

The application leaks information about username validity through differences in response processing time. A valid username causes the application to perform additional processing before returning the response, while an invalid username can be rejected more quickly. Because this difference is observable by an attacker, response timing becomes an unintended source of information disclosure.

The brute-force protection also relies on the client IP without properly accounting for spoofable or proxy-related headers such as `X-Forwarded-For`.
## Impact

An attacker can use response timing to identify valid usernames without needing to authenticate. Once a valid username is identified, it can be targeted with password attacks. If the application's brute-force protections can also be bypassed or circumvented, this can ultimately lead to account compromise.
## Remediation

Implement a Defense in Depth (DiD) approach to authentication that specifically addresses timing side-channels and spoofable request headers:

* **Mitigate Timing Discrepancies (Constant-Time Execution):** The application must process authentication requests in a uniform amount of time, regardless of whether the username exists. If an invalid username is submitted, the backend must still perform a computationally equivalent dummy operation (e.g., executing the password hashing algorithm against a dummy hash) before returning the response. This eliminates the time delta used for enumeration.
* **Secure IP Tracking for Rate Limiting:** Stop trusting user-controllable headers like `X-Forwarded-For` for security controls. Rate limiting and IP blocking must rely on the actual network-layer source IP (from the TCP connection). If the application sits behind a load balancer or reverse proxy, configure the proxy to strip client-supplied `X-Forwarded-For` headers and securely append the true client IP, ensuring the application only trusts headers originating from the proxy itself.
* **Account-Based Lockout (Defense in Depth):** In addition to IP-based rate limiting, implement account-level lockouts (e.g., locking a specific username after 5 failed attempts). This prevents brute-forcing the password even if the attacker successfully enumerates a valid username and bypasses IP controls.
* **Generic Error Messages:** Ensure the application returns identical, generic error messages (e.g., "Invalid username or password") for all failed login attempts to prevent basic logic-based enumeration.
