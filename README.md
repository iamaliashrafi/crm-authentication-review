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

## Session lifetime review · 24 November 2026

**D-01 · Configuration observation.** The project owner reviewed the local authentication configuration in `auth.ts` (line 78) and reported no custom session maximum age or separate short inactivity timeout. Auth.js documents a default session maximum age of 30 days. In the manager browser captures, the session cookie expiry moved from 24 December 09:57:49 GMT at sign-in to 24 December 10:00:33 GMT after dashboard access. This shows the browser cookie expiry was renewed during the observed session.

**Design confirmed by the project owner.** The CRM has no separate short idle sign-out. A session left unused is expected to remain available until its 30-day expiry measured from the last renewal. The application behavior at the expiry boundary has not been observed. The 30-second database connection setting applies only to the connection pool.

Implementation reference: [Auth.js session configuration](https://authjs.dev/reference/core#maxage).

## Coverage limits

The browser results apply to the manager flow in the local build. A fixed lifetime measured from the original sign-in, server-side invalidation of previously issued tokens, and authorization across roles remain unverified.

## Evidence handling

The public record contains request methods, route paths, response codes, and observed browser behavior. Credentials, cookie values, session tokens, account identifiers, and raw Burp captures are excluded.
