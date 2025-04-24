# Fetch retry

## Introduction


`fetch()` requests can fail due to transient network errors. Manual JavaScript retries are complex and impossible to be done after page unload (e.g. for `keepalive` fetches), causing data loss for critical beacons.

This proposal introduces a configurable, browser-managed retry mechanism within `fetch()`. It allows web developers to indicate that a `fetch()` request should be retried, to have a greater guarantee on it being reliably sent, even if the network is flaky. 

### Goals

-   Improve `fetch()` reliability on flaky networks, especially for `keepalive`.

-   Ensure retries occur correctly and in an efficient and controlled manner, even after page unload (when configured)

### Non-goals
-   Guarantee `fetch()` delivery. This feature only aims to increase probability of delivery, the request can still fail when we reach our retry limit.

-   Retry automatically without explicit opt-in for the cases that are ok to retry

## Proposed API

We propose adding a new `retryOptions` member to the `RequestInit` dictionary (the optional second argument to `fetch()`).

```idl
// Define the dictionary for retry configuration
dictionary  RetryOptions  {
  // Required: Maximum number of retry attempts after the initial one fails.
  unsigned  long  maxAttempts;

  // Optional: Delay before the first retry attempt (milliseconds).
  unsigned long  initialDelay;

  // Optional: Multiplier for increasing delay between retries (e.g., 2.0 for doubling).
  double  backoffFactor;

  // Optional: Maximum total time allowed for all retry attempts (milliseconds from the first failure).
  unsigned  long  maxAge;

  // Optional: Controls if retries can continue after document unload.
  // Requires `keepalive: true` on the fetch request to be effective.
  boolean  retryAfterUnload;
};

// Extend the existing RequestInit dictionary
partial  dictionary  RequestInit  {
  [SecureContext]  RetryOptions?  retryOptions;

};

// --- Example Usage ---

fetch("/api/important-beacon?id=12345",  {
  method:  "GET",
  keepalive:  true, // Essential for retryAfterUnload: true
  retryOptions:  {
    maxAttempts:  3,        // Max 3 retries (4 total attempts)
    initialDelay:  500,    // Start with 500ms delay
    backoffFactor:  2.0, // Double delay each time (500ms, 1s, 2s)
    maxAge:  60000,        // Give up after 60 seconds total retry time
    retryAfterUnload:  true  // Allow retries to continue even if page closes
  }
});

fetch("/api/logging,  {
  method:  "POST",
  body: data,
  keepalive:  true, // Essential for retryAfterUnload: true
  retryOptions:  {
    maxAttempts:  5,        // Max 5 retries (6 total attempts)
    retryNonIdempotent: true  // Required to allow retrying POST
    // Use dfault value for the retry delay etc.
  }
});
```

### API Details

`retryOptions` Object: A dictionary containing parameters to control the retry behavior. If omitted, no retries occur (current behavior).

-   `maxAttempts` (required): Specifies the maximum number of retry attempts after the initial attempt fails due to a retryable error. maxAttempts: 0 means no retries. Browsers must enforce a reasonable maximum limit (e.g., 5-10 per request, 20-30 per document) to prevent abuse.

-   `initialDelay`: Time in milliseconds before the first retry attempt.

-   `backoffFactor`: Multiplier applied to the delay for subsequent retries (e.g., 2.0 doubles the delay: `initialDelay`, `initialDelay * 2`, `initialDelay * 4`, ...). A factor of 1.0 means fixed delay. Note that browsers should implicitly apply randomization (jitter) to calculated delays to help prevent synchronized retries (thundering herd).


-   `maxAge`: An optional overall time limit in milliseconds, measured from the first failure, after which no further retry attempts will be made, regardless of maxAttempts.
-   `retryAfterUnload`:   Controls whether the browser should continue attempting retries even after the originating document (page/tab) has been unloaded.  Crucially, setting this to true requires keepalive: true to be set on the same `fetch()` call. The keepalive flag provides the mechanism for the request to outlive the document; `retryAfterUnload: true` leverages this to allow retries to also outlive the document.

-   If `keepalive: false`, setting `retryAfterUnload: true` will likely have no effect or be considered invalid, as the browser typically aborts standard requests on unload.

-   The default value could potentially be linked to the keepalive value (e.g., defaults to true if keepalive is true, false otherwise), or it could default to false requiring explicit opt-in. This needs discussion.

### Retry Behavior Details

- Retries will be attempted from the failed/latest redirect hop. If a `fetch()` request follows HTTP redirects (e.g., 301, 302, 307, 308), any necessary retries are performed against the URL right after the last successful redirect step, not the original URL provided to `fetch()`. For example, if fetch('/a') redirects to /b, and the request to /b subsequently fails with a network error, the retry attempts will target /b, not /a.

-   Retry count header will be sent. To allow servers to identify retry attempts (for logging, debugging, or deduplicating logic), each retry request initiated by the browser will include an additional HTTP header indicating the attempt number.

    -   Proposed Header:  `Retry-Attempt` (Exact name TBD during standardization).

    -   Value: An integer representing the current retry attempt number. The first retry would have `Retry-Attempt: 1`, the second `Retry-Attempt: 2`, and so on, up to the value specified in `maxAttempts`.

    -   The initial request (attempt 0) will not include this header.

-   Retries are intended solely for transient network errors where retrying the identical request might succeed. This typically includes errors at the TCP/IP level like connection timeouts, connection resets, connection refused (potentially), or DNS resolution failures if resolution previously succeeded for the host. For example, retries will not be triggered by:

    -   Successful HTTP responses, even with error status codes (4xx, 5xx). (Retrying on 5xx could be a future extension).

    -   Programmatic cancellation via `AbortSignal`.

    -   Security-related failures (CORS errors, CSP violations, mixed content blocks).

    -   Errors parsing request components (e.g., invalid URL).

-   We won't retry for non-idempotent method unless explicitly opted in:
    -   HTTP methods like GET, HEAD, OPTIONS, PUT, DELETE are generally idempotent (repeating the request has the same effect as making it once). Retrying these methods is generally safe.
    -   Methods like POST (and often PATCH) are non-idempotent. Automatically retrying a POST can lead to unintended consequences like creating duplicate resources or processing a transaction multiple times if the first request succeeded server-side but the response was lost due to network issues.
    -   Safety Proposal: To prevent accidental data corruption, the default behavior should restrict automatic retries to idempotent methods only.
    -   Enabling retries for non-idempotent methods like POST should require an explicit opt-in (e.g., a separate `retryNonIdempotent: true` flag within retryOptions or similar mechanism) or might be disallowed initially. Developers opting into retrying non-idempotent requests must ensure their server endpoints are designed to handle potential duplicates gracefully (e.g., using an `Idempotency-Key` header or checking the `Retry-Attempt` header).

-   Error Handling: The `fetch()` promise behaves as follows:
    -   If the initial attempt succeeds, the promise resolves with the `Response`.
    -   If the initial attempt fails but a subsequent retry succeeds, the promise resolves with the `Response` from the successful retry.
    -   If the initial attempt and all allowed retry attempts fail (due to retryable network errors, hitting `maxAttempts` limit, or exceeding `maxAge`), the promise rejects with the network error (`TypeError`) from the last attempt.

-   The initial proposal does not include a mechanism to expose detailed information about the retry process (e.g., number of attempts made, intermediate errors) back to the client-side JavaScript, although the `Retry-Attempt` header provides information to the server.

Existing Ways to Retry Fetches
------------------------------

1.  Manual JavaScript Retry Logic: Developers write `try...catch` blocks, manage `setTimeout` for delays (often implementing exponential backoff), and track attempt counts.
    -   Limitations: Requires boilerplate code, potentially complex to manage state correctly. Doesn't work for `keepalive` requests as the JavaScript context is unavailable to handle retries after page unload. This will also retry from the initial URL only, and not the last redirect hops as proposed, since redirects are invisible (except if the fetch user uses `redirect: manual`, but even so, the redirect URL can be opaque if it's cross-origin, and redirect handling can't work)

2.  Service Workers: Can intercept fetch events using an event listener. This allows for implementing custom, sophisticated retry logic, potentially including offline queueing.
    -   Limitations: Involves the complexity of Service Worker registration, lifecycle management, and communication. While powerful, it's significant overhead for simple retry needs. Reliably handling keepalive fetches intercepted just before unload requires careful SW design.

3.  Background Sync API: Allows deferring work until the browser detects stable network connectivity, managed via the Service Worker.
    -   Limitations: Designed for offline tolerance and synchronization, typically involving longer delays (minutes, with browser-controlled backoff) than desired for immediate retries of transient network errors. Not suitable for near-real-time beaconing scenarios where a quick retry is preferred.

 Security and Privacy Considerations
====================================

-   Resource Exhaustion: Malicious or misconfigured sites could attempt to trigger excessive retries, potentially impacting network resources or target servers. Mitigation relies on browsers enforcing strict, reasonable limits on `maxAttempts` and `maxAge`, alongside implementing backoff delays.

-   Idempotency Risks: The potential for unintended side effects when retrying non-idempotent methods is significant. Mitigation involves defaulting to only retrying idempotent methods and requiring explicit developer opt-in if non-idempotent retries are permitted.

-   Information Leakage (Retry-Attempt Header): The proposed `Retry-Attempt` header explicitly reveals the retry state of a request to the target server and any intermediaries. While useful for debugging and server deduplication logic, it does constitute information disclosure about the client's network behavior for that request. This seems acceptable given the feature's purpose but should be noted.

-   Timing Attacks/Information Leakage: The timing patterns of retry attempts could theoretically leak some information about network conditions. This is unlikely to provide substantially more information than can already be inferred by observing standard network request timings and failures. Additionally the browser will add random delays/jitters as well. The risk is considered low.

-   Standard Fetch Security: Each retry attempt must adhere to all standard web platform security policies, including CORS (preflights may need re-validation depending on timing/caching), CSP, credential handling, mixed content blocking, etc.

-   Retrying After Unload: Users might not expect that a fetch can run in the background after the initiator document has been unloaded. To mitigate this, we will only allow retry attempts when a fully active document with the same network isolation key as the initiator document exists. If a scheduled retry is triggered when there is no such document, it will wait until a document with the same network isolation key becomes fully active.

Potential extensions
====================

-   Configurable Retry Triggers: Allow opting into retries on specific HTTP 5xx server error status codes (e.g., `retryOnStatusCodes: [503, 504]`). This would need careful consideration due to potential server load implications.

-   Expose Retry Metadata to Client: Provide information back to the client-side JavaScript about the retry process, such as the final number of attempts made or the specific error on the last try, perhaps via properties on the resolved Response or the rejected Error.

-   Retry for HTML Elements: Explore extending a similar declarative mechanism to resource-loading HTML elements (`<img>`, `<link rel="stylesheet">`, `<script>`) to improve page resilience.
