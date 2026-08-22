
## About the lab

 **Difficulty:** Apprentice
 
 **Category:** Access Control Vulnerabilities
 
 **Lab URL:** [Lab: Unprotected admin functionality with unpredictable URL](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url)

## Vulnerability Summary

This lab has an unprotected admin panel. It's located at an unpredictable location, but the location is disclosed somewhere in the application.
Solve the lab by accessing the admin panel, and using it to delete the user carlos.
The app suffers from broken access control. The only defense mechanism implemented to conceal sensitive functionality is by making the admin URL unpredictable, however this unpredictability is easily locatable by public `HTML` source code of the page, allowing for vertical privilege escalation.
## Reconnaissance

Clicking on "View Page Source" to view the public HTML source code of the webpage, I see a `JavaScript` script that reads:
```javascript
var isAdmin = false; 
if (isAdmin) { 
	var topLinksTag = document.getElementsByClassName("top-links")[0]; 
	var adminPanelTag = document.createElement('a'); 
	adminPanelTag.setAttribute('href', '/admin-randomstring'); 
	adminPanelTag.innerText = 'Admin panel'; 
	topLinksTag.append(adminPanelTag); 
	var pTag = document.createElement('p'); pTag.innerText = '|'; 
	topLinksTag.appendChild(pTag); 
}
```
This basically means that if `isAdmin` is true, the page is dynamically modified to add a link labeled "Admin panel" pointing to the URL `url/admin-randomstring`. The `setAttribute` function is to set the link's destination URL, which suggests this might be the admin URL.
## Exploitation Steps

1. Access the lab. On the homepage, right click on a blank space, then select "View Page Source".
2. On the `view-source:url`, after scrolling down a bit, you should see a `JavaScript` script that reads something like this:
```javascript
var isAdmin = false; 
if (isAdmin) { 
	var topLinksTag = document.getElementsByClassName("top-links")[0]; 
	var adminPanelTag = document.createElement('a'); 
	adminPanelTag.setAttribute('href', '/admin-randomstring'); 
	adminPanelTag.innerText = 'Admin panel'; 
	topLinksTag.append(adminPanelTag); 
	var pTag = document.createElement('p'); pTag.innerText = '|'; 
	topLinksTag.appendChild(pTag); 
}
```
The random string differs from session to session.

3. Copy the `/admin-randomstring` and append it to the URL - navigate to `url/admin-randomstring`. You should see that there is an option to delete users `carlos` and `wiener`. Delete `carlos`, and the lab is solved.
## Payload Used

`/admin-randomstring`, `randomstring` found in the page source.
Sensitive functionalities are granted simply by reaching the admin URL. By finding out the hidden URL, a normal user or attacker can reach the admin URL and be granted admin's privileges.
## Root Cause

The main problem is that the application is treating the admin URL itself as a security control.
The admin panel is placed at an unpredictable URL, which might make it difficult for a normal user to find. However, the application does not actually check whether the person accessing that URL is authorized to use the admin functionality.
While investigating the application, I found that the supposedly hidden admin URL was exposed in the JavaScript source of the public page. This made the URL discoverable without needing any special privileges.
Once the URL was known, it could be accessed directly because there was no proper server-side authentication or authorization check to verify whether the current user had administrator privileges.
So the vulnerability is not simply that the admin URL was leaked. The bigger issue is that knowing the URL was enough to access functionality that should have been restricted to administrators.
In a properly secured application, discovering an admin endpoint should not automatically give a user access to it.
## Remediation

The application should not depend on hiding or making an admin URL difficult to guess as a security mechanism.
Instead, access to sensitive functionality should be controlled by the server.

A better approach would be:

Define which users or roles are allowed to access each sensitive endpoint.

Use proper role-based access control (RBAC) where appropriate.

Perform an authorization check on every request to the admin functionality.

Determine the user's permissions from their authenticated session rather than trusting information supplied by the client.

Deny the request when the user does not have the required privileges.
