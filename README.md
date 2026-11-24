# CRM Authentication Review

An educational case study of email-and-password sign-in and session handling in a CRM I built with Next.js and Prisma. I used Burp Suite to compare what the browser displayed with what the application returned over HTTP.

**Assessment status:** In progress. All three baseline checks passed for the manager account on 24 November 2026. The application supports four roles. The other three roles were outside this test run.

## Scope and environment

| Item | Assessment details |
| --- | --- |
| Target | My local CRM development instance at `http://localhost:3000` |
| Test account | Manager role, using the sign-in page at `/` and the dashboard at `/manager` |
| Tools | Burp Suite Proxy, HTTP history, and browser |
| Reference | [OWASP Top 10:2025 — A07 Authentication Failures](https://top10.owasp.org/2025/A07_2025-Authentication_Failures/) |
| Build | Application commit and build number were not recorded for this run |

## Objective and method

The question for this run was whether the browser could open the manager dashboard before sign-in, after a valid sign-in, and after sign-out. I used a local test account, submitted the sign-in form through the browser, and inspected the corresponding requests in Burp. I revisited the same protected route at each stage and recorded the page that actually loaded.

## Assessment coverage

| ID | Check | Expected behavior | Manager result |
| --- | --- | --- | --- |
| A-01 | Submit an incorrect password, then open the dashboard | The sign-in form shows an error and the dashboard requires sign-in | Passed |
| B-01 | Sign in with the correct password, then refresh the dashboard | The manager dashboard opens and remains available after refresh | Passed |
| C-01 | Sign out, then open the dashboard directly in the same browser | The browser returns to the sign-in page | Passed |

## Evidence journal · 24 November 2026

**A-01 · Rejected credentials.** The credentials callback returned HTTP 200 with `CredentialsSignin` in the response URL. The form displayed “Email or password is incorrect.” Opening `/manager` in the same browser returned the user to sign-in. The callback status alone would have been insufficient to classify the attempt.

**B-01 · Authenticated dashboard.** After a valid sign-in, the application issued a session cookie. The browser requested `GET /manager` and received HTTP 200 with the dashboard page. Refreshing the page produced another HTTP 200 and the dashboard remained visible.

**C-01 · Browser access after sign-out.** The browser submitted `POST /api/auth/signout` and received HTTP 200. Opening `/manager` directly then returned HTTP 302 with a location of `/`. The browser followed the redirect and loaded the sign-in page (HTTP 200). The dashboard request after sign-out carried no session cookie.

## Coverage limits and next check

These observations apply to the manager browser flow in the local build. Server-side invalidation of a previously issued session token, permissions across roles, and session expiration remain unverified.

The next check is to identify the intended idle and absolute session lifetimes in the CRM authentication configuration. The 30-second idle setting found so far governs database connections. The user-session timeout is still unknown.

## Evidence handling

The public record contains request methods, route paths, response codes, and observed browser behavior. Credentials, cookie values, session tokens, account identifiers, and raw Burp captures are excluded.
