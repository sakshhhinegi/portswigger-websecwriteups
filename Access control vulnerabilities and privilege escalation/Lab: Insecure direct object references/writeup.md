## About the Lab

 **Difficulty:** Apprentice

 **Category:** Access Control Vulnerabilities

 **Lab URL:** [Lab: Insecure direct object references](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)

## Vulnerability Summary

The application uses predictable filenames to identify chat transcripts and does not properly verify whether the current user is authorized to access a requested transcript.
This creates an Insecure Direct Object Reference (IDOR) vulnerability. By changing the filename in the transcript request, I was able to access another stored chat transcript.
## Reconnaissance

I entered the live chat without logging in and exchanged a message with the application.
After selecting View transcript, a transcript file named:

2.txt

was downloaded.
I intercepted the request in Burp Suite and noticed that the transcript filename was directly included in the request.
Since the filename looked like a simple sequential identifier, I tested whether changing it would return a different transcript.
Subsequently, every request to download chat logs is made with the header `GET /download-transcript/n.txt`, n is incremented by 1 starting from 2. This suggests that we can access any transcript by replacing n with another number.
## Exploitation Steps

1. Opened the live chat without authentication.

2. Sent a message and received a response.

3. Selected View transcript.

4. Observed that the transcript was retrieved as 2.txt.

5. Captured the transcript request in Burp Suite.

6. Changed the referenced file from 2.txt to 1.txt.

You should get a `200 OK` HTTP Response that, in the response body, reads:
```
CONNECTED: -- Now chatting with Hal Pline --
You: Hi Hal, I think I've forgotten my password and need confirmation that I've got the right one
Hal Pline: Sure, no problem, you seem like a nice guy. Just tell me your password and I'll confirm whether it's correct or not.
You: Wow you're so nice, thanks. I've heard from other people that you can be a right ****
Hal Pline: Takes one to know one
You: Ok so my password is password_string. Is that right?
Hal Pline: Yes it is!
You: Ok thanks, bye!
Hal Pline: Do one!
```

7. The application returned another user's transcript.

8. The transcript exposed carlos's password.

9. Used the credentials to log into the carlos account.
## Payload Used

`GET /download-transcript/1.txt HTTP/1.1`
This is a simple HTTP request to download the chat logs file named `1.txt`.
## Root Cause

The application relies on a predictable filename to identify each chat transcript but does not perform an authorization check before returning the requested file.
The server effectively trusts the object reference supplied by the client:
Requested file → Return file
Instead, it should verify that the authenticated user is actually authorized to access that specific transcript.
Because this check is missing, changing the filename allows access to other users' data.
## Impact

An attacker could manipulate the transcript identifier and access chat logs belonging to other users.
Depending on the information stored in those transcripts, this could expose sensitive personal information, credentials, session information, or other confidential data.
In this lab, the vulnerability exposed carlos's password, which allowed account compromise.
## Remediation

The application should not rely on predictable object references as a form of access control.

- Verify authorization for every transcript request.
- Ensure the requested transcript belongs to the currently authenticated user.
- Use server-side authorization checks before returning stored files.
- Avoid exposing predictable sequential identifiers where they are unnecessary.
- Treat changing an object identifier as an expected attack scenario and ensure it results in an authorization failure.
