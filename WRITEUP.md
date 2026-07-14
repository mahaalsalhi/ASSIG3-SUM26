# Assignment 3 Write-Up

## FIX 1 – SQL Injection (Authentication Bypass)

### Vulnerable code
The login query inserted the username and password directly into the SQL statement. Because of this, an attacker could enter SQL code instead of a normal username and change how the query worked.

### Payload used
```
curator' --
```

### What the payload did
I used the payload above as the username. The `--` commented out the password check, so the application only checked the username. This allowed me to log in as the **curator** account without entering the correct password.

### Fix
I changed the login query to use parameterized queries with `?` placeholders. This prevents the user's input from becoming part of the SQL command.

```javascript
const sql =
  `SELECT id, username FROM users
   WHERE username = ? AND password = ?`;

const user = get(sql, [username, password]);
```

### Why it works
Using parameterized queries separates the SQL command from the user's input. Even if someone enters SQL code, the database treats it as plain text instead of executing it.

### Limitation
This fix prevents SQL injection, but it does not protect against weak passwords or stolen login credentials.

---

## FIX 2 – SQL Injection (Data Extraction)

### Vulnerable code
The search page inserted the search term and sort value directly into the SQL query. This allowed an attacker to inject SQL commands and retrieve information from other database tables.

### Payload used
```
nothing%' UNION SELECT username, password, 'x', 'x' FROM users --
```

### What the payload did
The payload closed the original search query and added a `UNION SELECT` statement that returned usernames and passwords from the users table. This exposed sensitive information that should not have been accessible.

### Fix
I used parameterized queries for the search value and replaced the sort parameter with an allow-list of approved values.

```javascript
const sortOptions = {
  'created_at DESC': 'created_at DESC',
  species: 'species',
  title: 'title',
  'title DESC': 'title DESC',
};

const safeSort = sortOptions[sort] || 'created_at DESC';

const sql =
  `SELECT id, title, species, location
   FROM listings
   WHERE title LIKE ? OR species LIKE ?
   ORDER BY ${safeSort}`;
```

### Why it works
The search value is treated as data instead of SQL code, while the allow-list only allows approved sorting options. This prevents attackers from injecting SQL through the `ORDER BY` clause.

### Limitation
This fix prevents SQL injection, but proper authorization is still needed to control who can access sensitive information.

---

## FIX 3 – Reflected XSS

### Vulnerable code
The application displayed the search term directly in the HTML page without encoding it. This allowed an attacker to inject HTML and JavaScript into the page.

### Payload used
```html
<img src=x onerror=alert(document.domain)>
```

### What the payload did
The payload created an invalid image. When the image failed to load, the browser executed the JavaScript inside the `onerror` event.

### Fix
I encoded the search term before displaying it by using the `escapeHtml()` function.

```javascript
const heading =
`<h1>Search</h1>
<p class="note">Showing results for "${escapeHtml(q)}"</p>`;
```

### Why it works
The `escapeHtml()` function converts special HTML characters into safe text. Instead of executing the payload, the browser displays it as plain text.

### Limitation
This protects the search page, but every page that displays user input must also use proper output encoding.

---

## FIX 4 – Stored XSS and DOM XSS

### Vulnerable code
User comments and the shared note feature displayed user input as HTML. This allowed attackers to inject JavaScript that could execute whenever another user viewed the page.

### Payload used
```html
<img src=x onerror=alert(document.domain)>
```

### What the payload did
The payload attempted to inject malicious HTML into a comment and the shared note feature. Without proper protection, the browser would execute the JavaScript instead of displaying it as text.

### Fix
I encoded stored comments with `escapeHtml()` and replaced `innerHTML` with `textContent` in the shared note feature.

```javascript
<p class="comment-body">${escapeHtml(c.body)}</p>
```

```javascript
bannerEl.textContent = `📎 Shared note: ${note}`;
```

### Why it works
The `escapeHtml()` function prevents stored comments from being interpreted as HTML, and `textContent` displays the shared note as plain text instead of parsing it as HTML.

### Limitation
These fixes protect the comment system and shared notes, but every feature that displays user input should use the same secure output handling.

---

## FIX 5 – Cookie Flags and Content Security Policy

### Vulnerable code
The session cookie was missing important security flags, and the application did not send a Content Security Policy (CSP) header.

### Payload used
```
node exploits/05-hardening.mjs
```

### What the payload did
I ran the provided hardening exploit script. It checked whether the application was using secure cookie settings and a Content Security Policy. Before the fix, it reported that these protections were missing.

### Fix
I added the `HttpOnly` and `SameSite=Strict` cookie attributes and configured a Content Security Policy for every response.

```javascript
res.setHeader(
  'Set-Cookie',
  `sid=${token}; Path=/; HttpOnly; SameSite=Strict`
);
```

```javascript
res.setHeader(
  'Content-Security-Policy',
  "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; object-src 'none'; base-uri 'none'; frame-ancestors 'none'"
);
```

### Why it works
`HttpOnly` prevents JavaScript from reading the session cookie, `SameSite=Strict` helps protect against cross-site request attacks, and the Content Security Policy restricts which resources the browser is allowed to load and execute.

### Limitation
These browser security features add another layer of protection, but they cannot replace secure coding practices or completely prevent every possible attack.
