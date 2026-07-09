---
title: Use Web Crypto for Postman signing scripts
tags: [postman, signature, webcrypto, harness]
scope: cross-project
status: draft
---

# Use Web Crypto for Postman signing scripts

## Context
When a Postman pre-request script must generate HMAC request signatures, the script needs a crypto API that is supported by the current Postman sandbox and can run before the request is sent.

## Wrong Direction
Using `crypto-js` examples copied from old snippets can fail or become brittle because current Postman documentation marks `crypto-js` as deprecated and directs scripts toward Web Crypto objects.

## Correct Pattern
Implement HMAC signing with `crypto.subtle.importKey` and `crypto.subtle.sign`, and use `await` so the headers are computed before calling `pm.request.headers.upsert`. Resolve Postman variables in the request URL with `pm.variables.replaceIn(pm.request.url.toString())` before canonicalizing the URL for signing.

## Reusable Insight
Postman signing scripts should be treated as small platform-specific ports: use the sandbox's supported crypto API, avoid duplicate headers by using `upsert`, and preserve the exact canonicalization rules from the backend/reference signer.

## Detection
Look for pre-request scripts that import or assume `crypto-js`, generate signatures asynchronously without `await`, add repeated `Authorization` headers with `add`, or sign unresolved URLs that still contain `{{variable}}` placeholders.

## Verification
Run the signing function with fixed credentials, timestamp, method, and URL from the reference implementation. Confirm that the generated `Authorization` and date headers match golden vectors, including raw-path percent-encoding cases.
