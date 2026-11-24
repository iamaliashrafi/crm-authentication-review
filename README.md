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

## Sign-in error consistency · 24 November 2026

**E-01 · Passed for the two sampled sign-in attempts.** An existing manager account with an incorrect password and an unregistered test address both received HTTP 200 from the credentials callback. Both JSON responses returned the same error parameters, `error=CredentialsSignin&code=credentials`. The browser displayed “Email or password is incorrect.” in both cases. The status, error payload, and visible message did not distinguish the two inputs. Response timing and other account flows were outside this check.

## Sign-in rate limiting · Source review

**F-01 · Helper logic confirmed from an owner-supplied excerpt.** The application stores a counter per normalized email address in the database. The first call starts a fixed 15-minute window. Calls 1 through 10 return true. Call 11 and subsequent calls return false until the window expires. The first call after expiry starts a new window at 1. The helper counts invocations and receives no password-validation result. Integration with the sign-in handler, any reset after successful authentication, and the separate IP limit mentioned in a source comment remain unverified.

**F-02 · Black-box observation (24 November 2026).** During a short manual series of invalid sign-ins, the tester reported the same generic form message throughout and no change in the HTTP status or returned error between the first and last attempts. A follow-up sign-in with the correct password succeeded: the credentials callback issued a session cookie, and the browser loaded the manager dashboard (GET /manager, HTTP 200). The account could sign in from that browser at the time of the follow-up. No throttling signal was observed in this sample. The activation threshold and runtime enforcement remain unverified.

## Manager sign-in and MFA · 24 November 2026

**G-01 · No second factor requested in the observed manager sign-in.** The tester confirmed that entering the correct email and password led directly to the manager dashboard. Burp showed a session cookie issued by the credentials callback, followed by a successful dashboard request. This observation covers the tested account, browser and local deployment. Enforcing MFA for manager accounts would add an independent check before granting dashboard access. [OWASP A07 prevention guidance](https://top10.owasp.org/2025/A07_2025-Authentication_Failures/)

## Password recovery · 24 November 2026

**H-01 · Sign-in form observation.** The project owner confirmed that the local sign-in form intentionally omits a self-service password recovery option. A Superadmin selects and saves a new password for the user in the administrative panel.

**H-02 · Administrative password reset (passed for fresh sign-ins).** After the Superadmin changed the local test account’s password, the old password was rejected. Burp showed HTTP 200 with `CredentialsSignin` in the callback response. The tester confirmed successful sign-in with the new password in the browser. Access by other roles to the reset function remains unverified.

**H-03 · Existing browser session after password reset (24 November 2026).** The tester refreshed the dashboard in a browser session that had been open before the Superadmin reset the test account’s password. The browser returned to the sign-in form. This result covers the single observed session.

## Password policy in administrative resets · 24 November 2026

**I-01 · Short password rejected (passed for the tested input).** The tester confirmed that the Superadmin reset form rejected the short password. During the short-password check in the Superadmin reset form, the tester reported the validation message `password must be at least 8 characters and at most 72 UTF-8 bytes`. The message states a minimum of 8 characters and a maximum of 72 encoded bytes. Acceptance at either boundary and the layer enforcing the rule remain unverified.

## Coverage limits

The browser results apply to the manager flow in the local build. A fixed lifetime measured from the original sign-in, server-side invalidation of previously issued tokens, and authorization across roles remain unverified.

## Evidence handling

The public record contains request methods, route paths, response codes, and observed browser behavior. Credentials, cookie values, session tokens, account identifiers, and raw Burp captures are excluded.
