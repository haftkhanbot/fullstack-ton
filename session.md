do same for
```typescript

import { jwtVerify, SignJWT } from 'jose'
import { cookies } from 'next/headers'
import { NextRequest, NextResponse } from 'next/server'

const key = new TextEncoder().encode(process.env.JWT_SECRET)

export const SESSION_DURATION = 60 * 60 * 1000 // 1 hour

export async function encrypt(payload: any) {
    return await new SignJWT(payload)
        .setProtectedHeader({ alg: "HS256" })
        .setIssuedAt()
        .setExpirationTime("1 hour")
        .sign(key)
}

export async function decrypt(input: string): Promise<any> {
    const { payload } = await jwtVerify(input, key, {
        algorithms: ["HS256"],
    })
    return payload
}

export async function getSession() {
    const session = cookies().get("session")?.value
    console.log("Session value in getSession ", session)
    if (!session) return null
    return await decrypt(session)
}

export async function updateSession(request: NextRequest) {
    const session = request.cookies.get("session")?.value
    if (!session) return

    // Refresh the session so it doesn't expire
    const parsed = await decrypt(session)
    parsed.expires = new Date(Date.now() + SESSION_DURATION)
    const res = NextResponse.next()
    res.cookies.set({
        name: "session",
        value: await encrypt(parsed),
        httpOnly: true,
        expires: parsed.expires,
    })
    return res
}
```
Okay, let's break down this code snippet, which focuses on handling session management using JWTs and cookies within a Next.js application. We'll analyze each function and its purpose:

**Overall Purpose:**

This code implements a secure session management system using JSON Web Tokens (JWTs) stored in HTTP cookies. It allows you to:

*   **Encrypt data** to create JWT session tokens.
*   **Decrypt JWT session tokens** to retrieve user session information.
*   **Store sessions** in cookies.
*   **Retrieve sessions** from cookies.
*   **Refresh/Extend sessions** to prevent them from expiring too soon.

**Code Breakdown:**

1.  **Imports and Constants:**

    ```typescript
    import { jwtVerify, SignJWT } from 'jose'
    import { cookies } from 'next/headers'
    import { NextRequest, NextResponse } from 'next/server'

    const key = new TextEncoder().encode(process.env.JWT_SECRET)
    export const SESSION_DURATION = 60 * 60 * 1000 // 1 hour
    ```

    *   `import { jwtVerify, SignJWT } from 'jose'`: Imports functions from the `jose` library for JWT creation, signing, and verification.
    *   `import { cookies } from 'next/headers'`: Imports the `cookies` object from Next.js, used to access and modify HTTP cookies.
    *   `import { NextRequest, NextResponse } from 'next/server'`: Imports types for request and response objects used in Next.js server-side functions.
    *   `const key = new TextEncoder().encode(process.env.JWT_SECRET)`: Creates a cryptographic key from the environment variable `JWT_SECRET` using `TextEncoder` to convert a string into a `Uint8Array` which is required by `jose`. This key is crucial for securely signing and verifying JWTs.
    *   `export const SESSION_DURATION = 60 * 60 * 1000`: Defines the default session duration as 1 hour in milliseconds.

2.  **`encrypt` function:**

    ```typescript
    export async function encrypt(payload: any) {
        return await new SignJWT(payload)
            .setProtectedHeader({ alg: "HS256" })
            .setIssuedAt()
            .setExpirationTime("1 hour")
            .sign(key)
    }
    ```

    *   `export async function encrypt(payload: any)`: An asynchronous function to encrypt data into a JWT token. It takes a `payload` (an object or any value that you want to store in the session) as an argument.
    *   `return await new SignJWT(payload)`: Creates a new JWT signing object with the given payload.
    *   `.setProtectedHeader({ alg: "HS256" })`: Sets the signing algorithm to HMAC SHA256 ("HS256"). This is a symmetric signing algorithm where the same key is used for signing and verifying the JWT.
    *   `.setIssuedAt()`: Sets the `iat` (issued at) claim in the JWT, indicating when the token was created.
    *   `.setExpirationTime("1 hour")`: Sets the `exp` (expiration time) claim in the JWT, setting it to 1 hour from the issued at time by default.
    *   `.sign(key)`: Signs the JWT with the `key`, creating the final JWT string.

3.  **`decrypt` function:**

    ```typescript
    export async function decrypt(input: string): Promise<any> {
        const { payload } = await jwtVerify(input, key, {
            algorithms: ["HS256"],
        })
        return payload
    }
    ```

    *   `export async function decrypt(input: string): Promise<any>`: An asynchronous function to decrypt (verify) a JWT token. It takes the JWT string as `input` and returns the original payload.
    *   `const { payload } = await jwtVerify(input, key, { algorithms: ["HS256"] })`: Verifies the JWT against the secret `key` using `jwtVerify` and confirms that the token was signed using `HS256`. If the signature is valid, it extracts the payload. If verification fails, it will throw an error.
    *   `return payload`: Returns the decrypted payload from the JWT.

4.  **`getSession` function:**

    ```typescript
    export async function getSession() {
        const session = cookies().get("session")?.value
        console.log("Session value in getSession ", session)
        if (!session) return null
        return await decrypt(session)
    }
    ```

    *   `export async function getSession()`: An asynchronous function to retrieve and decrypt the session from the cookies.
    *   `const session = cookies().get("session")?.value`: Retrieves the value of the "session" cookie. The `?.value` handles cases where the cookie is not found.
    *   `console.log("Session value in getSession ", session)`: Logs the session value to the console (useful for debugging).
    *   `if (!session) return null`: Returns null if the session cookie is not found (no user logged in).
    *   `return await decrypt(session)`: Decrypts the session value using the `decrypt` function and returns the session payload.

5.  **`updateSession` function:**

    ```typescript
    export async function updateSession(request: NextRequest) {
        const session = request.cookies.get("session")?.value
        if (!session) return

        // Refresh the session so it doesn't expire
        const parsed = await decrypt(session)
        parsed.expires = new Date(Date.now() + SESSION_DURATION)
        const res = NextResponse.next()
        res.cookies.set({
            name: "session",
            value: await encrypt(parsed),
            httpOnly: true,
            expires: parsed.expires,
        })
        return res
    }
    ```

    *   `export async function updateSession(request: NextRequest)`: An asynchronous function to update the session cookie and its expiry.
    *   `const session = request.cookies.get("session")?.value`: Retrieves the session from the request cookies.
    *   `if (!session) return`: If the session is not present (no cookie), returns early.
    *   `const parsed = await decrypt(session)`: Decrypts the session using the `decrypt` function.
    *   `parsed.expires = new Date(Date.now() + SESSION_DURATION)`: Updates the `expires` field in the decrypted session payload by adding the `SESSION_DURATION` to the current time.
    *   `const res = NextResponse.next()`: Creates a new response using `NextResponse.next()`. This will pass control to the next handler.
    *    `res.cookies.set({ ... })`: Updates the `session` cookie in the response.
        *   `name: "session"`: Sets the cookie name to "session."
        *   `value: await encrypt(parsed)`: Encrypts the updated `parsed` session payload with the current expiry.
        *   `httpOnly: true`: Sets the httpOnly flag on the cookie so that javascript on the client-side can not access this cookie, adding extra security.
        *   `expires: parsed.expires`: Set the expiration date of the cookie as the updated date value.
    *   `return res`: Returns the updated response to the client including the modified cookies.

**Key Points:**

*   **JWTs for Session Management:** JWTs are used to create stateless sessions, eliminating the need to store session data on the server. All the session data is encrypted within the JWT.
*   **Cookies for Storage:** The encrypted JWT is stored in an HTTP cookie, which allows the browser to send the token with every request, maintaining user sessions.
*   **Security:**  The `JWT_SECRET` and the `HS256` algorithm are used for securely signing and verifying the JWTs, ensuring the integrity and authenticity of the session data.
*   **Session Refreshing:** The `updateSession` function prevents the user from being logged out abruptly by extending the session's expiration time on each request. This implements session sliding.
*   **Next.js Integration:** The code leverages the `cookies` and `NextRequest`/`NextResponse` from Next.js, making it easy to work within a Next.js application.

**How to Use this:**

1.  **`encrypt` for login:** After a successful login, use the `encrypt` function to generate a JWT containing user data.
2.  **Set the cookie:** In your API route response, set the `session` cookie with the generated JWT using `res.cookies.set`.
3.  **`getSession` for user retrieval:** In your server-side components or API routes, use `getSession` to retrieve the user session information from the cookie.
4.  **`updateSession` for session refresh:** Use `updateSession` as a middleware in each request to ensure that user sessions don't expire prematurely.

**Example Usage (Conceptual):**

```typescript
// In an API route (e.g., app/api/login/route.ts)

import { encrypt } from '../utils/session' // Assume session utils file path
import { NextResponse } from 'next/server'

export async function POST(req) {
  // ... perform user authentication ...

  const user = { id: 'user123', name: 'John Doe' } // Example user data
  const sessionToken = await encrypt(user)

  const res = NextResponse.json({ message: 'Logged in' }, { status: 200 })
  res.cookies.set({
    name: 'session',
    value: sessionToken,
    httpOnly: true,
    expires: new Date(Date.now() + 60 * 60 * 1000) // Set session to expire in 1 hour
  })

  return res
}

// In a server-side component (e.g., app/dashboard/page.tsx)
import { getSession } from '../utils/session'

export default async function Dashboard() {
    const session = await getSession()

    if (!session) {
        return <p>Not Logged In</p>
    }
  return <p>Welcome, {session.name}</p>
}
```
This detailed explanation should give you a solid understanding of how the provided session management code works within a Next.js application using JWTs and cookies. Remember that storing the `JWT_SECRET` correctly is crucial for security (e.g., in an environment variable).


https://github.com/nikandr-surkov/Make-TON-Telegram-Mini-App-2/blob/master/utils/session.ts
