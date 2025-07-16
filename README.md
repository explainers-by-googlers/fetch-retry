# Fetch retry

## Introduction

`fetch()` requests can fail due to transient network errors. Manual JavaScript retries are complex and impossible to be done after page unload (e.g. for `keepalive` fetches), causing data loss for critical beacons.

This proposal introduces a configurable, browser-managed retry mechanism within `fetch()`. It allows web developers to indicate that a `fetch()` request should be retried, to have a greater guarantee on it being reliably sent, even if the network is flaky. 

### Goals

-   Improve `fetch()` reliability on flaky networks, especially for `keepalive`.
-   Ensure retries occur correctly and in an efficient and controlled manner, even after page unload (when configured)

### Non-goals
-   Guarantee `fetch()` delivery. This feature only aims to increase probability of delivery, the request can still fail when we reach our retry limit.
-   Retry automatically without explicit opt-in for what cases that are ok to retry

## Proposed API

We propose adding a new `retryOptions` member to the `RequestInit` dictionary (the optional second argument to `fetch()`).

```webidl
// Define the dictionary for retry configuration
dictionary RetryOptions {
  // Required: Maximum number of retry attempts after the initial one fails.
  [EnforceRange] required unsigned short maxAttempts;

  // Optional: Delay before the first retry attempt (milliseconds).
  [EnforceRange] unsigned long long initialDelay;

  // Optional: Multiplier for increasing delay between retries (e.g., 2.0 for doubling).
  double backoffFactor;

  // Maximum total time allowed for all retry attempts (milliseconds from initial request start).
  [EnforceRange] unsigned long long maxAge;

  // Optional: Controls if retries can be attempted after document unload.
  // Requires `keepalive: true` on the fetch request to be effective.
  //  Defaults to false.
  boolean retryAfterUnload;

  // Optional: Specifies whether to retry when the HTTP request method is non-idempotent (e.g. POST).
  // If this is not set while the HTTP request method of the fetch is non-idempotent, no retry will be attempted.
  // Defaults to false.
  boolean retryNonIdempotent;
};

// Extend the existing RequestInit dictionary
partial  dictionary  RequestInit {
  [SecureContext] RetryOptions retryOptions;

};
```
```js
// --- Example Usage ---

fetch("/api/important-beacon?id=12345",  {
  method: "GET",
  keepalive: true, // Essential for retryAfterUnload: true
  retryOptions:  {
    maxAttempts: 3,        // Max 3 retries (4 total attempts)
    initialDelay: 500,    // Start with 500ms delay
    backoffFactor: 2.0, // Double delay each time (500ms, 1s, 2s)
    maxAge: 60000,        // Give up after 60 seconds total retry time
    retryAfterUnload: true  // Allow retries to continue even if page closes
  }
});

fetch("/api/logging",  {
  method: "POST",
  body: data,
  keepalive: true, // Essential for retryAfterUnload: true
  retryOptions: {
    maxAttempts: 5,        // Max 5 retries (6 total attempts)
    retryNonIdempotent: true  // Required to allow retrying POST
    // Use default value for the retry delay etc.
  }
});
```

### API Details

`retryOptions` Object: A dictionary containing parameters to control the retry behavior. If omitted, no retries occur (current behavior).
-   `maxAttempts` (required): Specifies the maximum number of retry attempts after the initial attempt fails due to a retryable error. maxAttempts: 0 means no retries. Browsers must enforce a reasonable maximum limit (e.g., 5-10 per request, 20-30 per document) to prevent abuse.
-   `initialDelay`: Time in milliseconds before the first retry attempt.
-   `backoffFactor`: Multiplier applied to the delay for subsequent retries (e.g., 2.0 doubles the delay: `initialDelay`, `initialDelay * 2`, `initialDelay * 4`, ...). A factor of 1.0 means fixed delay. Note that browsers should implicitly apply randomization (jitter) to calculated delays to help prevent synchronized retries (thundering herd).
-   `maxAge`: An optional overall time limit in milliseconds, measured from the first failure, after which no further retry attempts will be made, regardless of maxAttempts.
-   `retryAfterUnload`:   Controls whether the browser should continue attempting retries even after the originating document (page/tab) has been unloaded, but only if a same-network-isolation-key document is active in the same browsing session.  Crucially, setting this to true requires keepalive: true to be set on the same `fetch()` call. The keepalive flag provides the mechanism for the request to outlive the document; `retryAfterUnload: true` leverages this to allow retries to also outlive the document.
    -   If `keepalive: false`, setting `retryAfterUnload: true` will likely have no effect or be considered invalid, as the browser typically aborts standard requests on unload.
    -   **Important point for privacy**: Even though this allows retry after the original document is unloaded, it requires that a document with the same network isolation key as the original initiator of the fetch is active in the same browsing session. If there is no such document, a retry will not be attempted, and it will wait until such a document becomes active (e.g. through navigation). This is important because we don't want the retry network requests to leak information about the network that the user is on. If the user has an active document with the same network isolation key, this is not a problem, since that document itself is able to initiate a similar fetch.
- `retryNonIdempotent`: Required to be set to true for non-idempotent HTTP methods for the retry to actually happen

### Retry Behavior Details

- Retries will be **attempted from the original URL & fetch params**. If a `fetch()` request follows HTTP redirects (e.g., 301, 302, 307, 308), any necessary retries are performed against the original URL & fetch params provided to `fetch()`. For example, if fetch('/a') redirects to `/b`, and the request to `/b` subsequently fails with a network error, the retry attempts will target `/a`, not `/b`.
-   Retries are intended solely for **transient network errors** where retrying the identical request might succeed. This typically includes errors at the TCP/IP level like connection timeouts, connection resets, connection refused (potentially), or DNS resolution failures if resolution previously succeeded for the host. For example, retries will not be triggered by:
    -   Successful HTTP responses, even with error status codes (4xx, 5xx).
    -   Programmatic cancellation via `AbortSignal`.
    -   Security-related failures (CORS errors, CSP violations, mixed content blocks).
-  However, to prevent timing attacks that leaks information about whether a network error is policy related (which will not be retried) or not (which might be retried), all network errors will only be surfaced when the navigation reaches the max age. This makes it impossible to tell from the script whether a retry have been attempted or not, since the fetch promise will be rejected at the same timing. See also [thread](https://github.com/whatwg/fetch/issues/1838#issuecomment-3069011754).
-   We **won't retry for non-idempotent method unless explicitly opted in**:
    -   HTTP methods like GET, HEAD, OPTIONS, PUT, DELETE are generally idempotent (repeating the request has the same effect as making it once). Retrying these methods is generally safe.
    -   Methods like POST (and often PATCH) are non-idempotent. Automatically retrying a POST can lead to unintended consequences like creating duplicate resources or processing a transaction multiple times if the first request succeeded server-side but the response was lost due to network issues.
    -   Safety Proposal: To prevent accidental data corruption, the default behavior should restrict automatic retries to idempotent methods only.
    -   Enabling retries for non-idempotent methods like POST should require an explicit opt-in (e.g., a separate `retryNonIdempotent: true` flag within retryOptions or similar mechanism). Developers opting into retrying non-idempotent requests must ensure their server endpoints are designed to handle potential duplicates gracefully (e.g., using an `Idempotency-Key` header or checking the `Retry-Attempt` header).
-   **Only the final result will be exposed**. The `fetch()` promise behaves as follows:
    -   If the initial attempt succeeds, the promise resolves with the `Response`.
    -   If the initial attempt fails but a subsequent retry succeeds, the promise resolves with the `Response` from the successful retry.
    -   If the initial attempt and all allowed retry attempts fail (due to retryable network errors, hitting `maxAttempts` limit, or exceeding `maxAge`), the promise rejects with the network error (`TypeError`) from the last attempt. Note that this might happen after the initiator document is unloaded if it's a `keepalive` request, so the initiator script might never know the final result (this is already a possibility even without retry).
-   The initial proposal does not include a mechanism to expose detailed information about the retry process (e.g., number of attempts made, intermediate errors) back to the client-side JavaScript, although the `Retry-Attempt` header provides information to the server.

 Security and Privacy Considerations
====================================

-   Resource Exhaustion: Malicious or misconfigured sites could attempt to trigger excessive retries, potentially impacting network resources or target servers. Mitigation relies on browsers enforcing strict, reasonable limits on `maxAttempts` and `maxAge`, alongside implementing backoff delays.
-   Idempotency Risks: The potential for unintended side effects when retrying non-idempotent methods is significant. Mitigation involves defaulting to only retrying idempotent methods and requiring explicit developer opt-in if non-idempotent retries are permitted.
-   Information Leakage (Retry-Attempt Header): The proposed `Retry-Attempt` header explicitly reveals the retry state of a request to the target server and any intermediaries. While useful for debugging and server deduplication logic, it does constitute information disclosure about the client's network behavior for that request. This seems acceptable given the feature's purpose but should be noted.
-   Timing Attacks/Information Leakage: The timing patterns of retry attempts could theoretically leak some information about network conditions. This is unlikely to provide substantially more information than can already be inferred by observing standard network request timings and failures. Additionally the browser will add random delays/jitters as well. The risk is considered low.
-   Standard Fetch Security: Each retry attempt must adhere to all standard web platform security policies, including CORS (preflights may need re-validation depending on timing/caching), CSP, credential handling, mixed content blocking, etc.
-   Retrying After Unload: Users might not expect that a fetch can run in the background after the initiator document has been unloaded. To mitigate this, we will only allow retry attempts when a fully active document with the same network isolation key as the initiator document exists. If a scheduled retry is triggered when there is no such document, it will wait until a document with the same network isolation key becomes fully active.

Appendix: Existing Ways to Retry Fetches
========================================
1.  Manual JavaScript Retry Logic: Developers write `try...catch` blocks, manage `setTimeout` for delays (often implementing exponential backoff), and track attempt counts.
    -   Limitations: Requires boilerplate code, potentially complex to manage state correctly. Doesn't work for `keepalive` requests as the JavaScript context is unavailable to handle retries after page unload.
2.  Service Workers: Can intercept fetch events using an event listener. This allows for implementing custom, sophisticated retry logic, potentially including offline queueing.
    -   Limitations: Involves the complexity of Service Worker registration, lifecycle management, and communication. While powerful, it's significant overhead for simple retry needs. Reliably handling keepalive fetches intercepted just before unload requires careful SW design.
3.  Background Sync API: Allows deferring work until the browser detects stable network connectivity, managed via the Service Worker.
    -   Limitations: Designed for offline tolerance and synchronization, typically involving longer delays (minutes, with browser-controlled backoff) than desired for immediate retries of transient network errors. Not suitable for near-real-time beaconing scenarios where a quick retry is preferred.


Appendix: Difference with other APIs
=====================================
In [discussions](https://github.com/whatwg/fetch/issues/1838#issuecomment-3020408525), a question came up on how this is different from [Extended Lifetime SharedWorkers](https://gist.github.com/domenic/c5bd38339f33b49120ae11b3b4af5b9b).
Both proposals are around trying to minimize "data loss of critical beacons", but they are tackling different problems. The Extended Lifetime SharedWorkers proposal is more about "we need to run some arbitrary operations after unload, with stricter bounded time":
- It can run arbitrary operations such as writing to storage, etc.
- It will only run once, and needs to stop quite soon after the document unload (compared to fetch retry)
- Like mentioned in this [section](this section), they're also useful when async steps are required

Meanwhile fetch retry is about "Try to ensure that this fetch gets sent, even if it takes a while":
- It's specific to fetches
- It's meant to make the fetches more resilent to potentially transient errors, which are actually quite common.
- The retry can be triggered quite a bit after the document is unloaded, but in such a way that isn't a problem privacy wise (only retrying when a same-NetworkIsolationKey document is committed).
- The retry can also be attempted when the document is still around

So the former is more around "making sure an operation is run, after potentially some async work" while the latter is more "fetches are more resilient to transient errors and have a higher chance of reaching the server". They can work together too, e.g. we can maybe trigger a fetch with retry from the worker and that makes sure it's attempted and with a higher chance of it actually getting through.
