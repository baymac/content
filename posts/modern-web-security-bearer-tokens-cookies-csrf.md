---
title: 'Modern Web Security: Bearer Tokens, Cookies, and CSRF'
date: '2025-07-31'
tags: web-security, authentication, csrf, web-development, security
---

Building secure web applications is a journey full of subtle pitfalls and best practices. Recently, I dove into authentication and CSRF (Cross-Site Request Forgery) protection while building a modern web app with both browser and mobile clients. Here's what I learned about keeping your applications secure.

## Bearer Tokens vs. Cookies for Authentication

### Bearer Tokens
- Sent in the `Authorization` header: `Authorization: Bearer <token>`
- Ideal for mobile, desktop apps, and SPAs
- Not sent automatically by browsers - must be set explicitly
- Immune to CSRF attacks (malicious sites can't set custom headers)

### Cookies
- Automatically sent by browsers with every request
- Can be secured with `HttpOnly` flag
- Vulnerable to CSRF attacks if not properly protected
- Browsers will send cookies even for malicious cross-site requests

## Understanding CSRF and Its Impact

**CSRF (Cross-Site Request Forgery)** occurs when a malicious website tricks a user's browser into making unwanted requests to your backend. If your application only checks for valid cookies, an attacker could perform actions on behalf of your users without their knowledge.

## How CSRF Protection Works

The standard defense is the **double-submit cookie** pattern:

1. Server sets a CSRF token cookie (not `HttpOnly`)
2. Frontend JavaScript reads this token and sends it in a custom header (e.g., `X-CSRF-Token`)
3. Backend verifies the token in the header matches the cookie value

This works because while browsers automatically send cookies, only your legitimate frontend can read the CSRF token and include it in the request headers.

## When Is CSRF Protection Necessary?

- **Cookies for authentication?**
  - ✅ Required for all state-changing endpoints (POST, PUT, DELETE, etc.)
  - Implement CSRF tokens for every form and AJAX request

- **Bearer tokens in headers?**
  - ❌ Not needed
  - Browsers can't set `Authorization` headers on cross-origin requests

## Best Practices

1. **For APIs and mobile/desktop apps**:
   - Use Bearer tokens in the `Authorization` header
   - No CSRF protection needed

2. **For traditional web apps**:
   - Use cookies for authentication
   - Implement CSRF protection for all state-changing endpoints
   - Set cookies as `SameSite=Lax` or `Strict`
   - Consider `HttpOnly` flag for sensitive cookies

3. **CSRF Tokens**:
   - Must be readable by JavaScript (not `HttpOnly`)
   - Should be unique per user session
   - Include in custom headers, not URL parameters

## Implementation Example

**Frontend (JavaScript):**
```javascript
// Read CSRF token from cookie and include in request
fetch('/api/sensitive-action', {
  method: 'POST',
  credentials: 'include',
  headers: {
    'X-CSRF-Token': getCSRFTokenFromCookie(),
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ action: 'update' })
});
```

**Backend (Go example):**
```go
func CSRFMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Skip CSRF check for safe methods
        if r.Method == http.MethodGet || r.Method == http.MethodHead || 
           r.Method == http.MethodOptions || r.Method == http.MethodTrace {
            next.ServeHTTP(w, r)
            return
        }

        csrfCookie, err := r.Cookie("csrf_token")
        if err != nil {
            http.Error(w, "CSRF token missing", http.StatusForbidden)
            return
        }

        csrfHeader := r.Header.Get("X-CSRF-Token")
        if csrfHeader == "" || csrfHeader != csrfCookie.Value {
            http.Error(w, "Invalid CSRF token", http.StatusForbidden)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

## Key Security Takeaways

1. **Bearer tokens** are ideal for APIs and client-side applications
2. **Cookies** work well for web but require CSRF protection
3. **CSRF tokens** must be implemented correctly to be effective
4. Always use **HTTPS** to protect tokens in transit
5. Keep your authentication logic simple and well-tested

## Final Thoughts

Security is not a feature you can bolt on at the end - it needs to be part of your application's foundation. By understanding these concepts and implementing them correctly, you'll be well on your way to building more secure web applications.

Remember: The best security is layered security. Combine these techniques with other security measures like rate limiting, input validation, and regular security audits for maximum protection.

*This article was generated with the assistance of Claude 4. Always review and adapt security practices to your specific use case and requirements.*
