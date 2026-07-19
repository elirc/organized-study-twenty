# Security Checklist

Threats mapped to this repo's actual defenses, then a pre-merge checklist. Learn the mapping; recite it in interviews with anchors.

| Threat | Defense here | Anchor | Notes |
| --- | --- | --- | --- |
| Broken authn | Single guard resolves all principals (user/API key/application) and binds context once | [jwt-auth.guard.ts#L23-L49](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L23-L49) | Many SSO providers, each isolated in its own guard |
| IDOR / cross-tenant | Structural: workspace-scoped datasource; schema-per-tenant | [common-base-query-runner.service.ts#L144-L153](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L144-L153) | The id in the URL can't reach another tenant's schema |
| Broken authz (in-tenant) | Object grants + join validation + row predicates + field restrictions in the ORM | [workspace-select-query-builder.ts#L342-L390](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L342-L390) | Audit every `shouldBypassPermissionChecks` |
| SQL injection | ORM-built queries; parser mediates user filters into TypeORM expressions | [graphql-query.parser.ts](../../../packages/twenty-server/src/engine/api/graphql/graphql-query-runner/graphql-query-parsers/graphql-query.parser.ts) | Danger zone: any raw `query()` with interpolated identifiers (see Kata 2) |
| SSRF | Centralized egress client blocks private/internal IPs for user-supplied URLs | [call-webhook.job.ts#L72-L88](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L72-L88), [secure-http-client.service.ts](../../../packages/twenty-server/src/engine/core-modules/secure-http-client/secure-http-client.service.ts) | Any new user-URL fetch MUST use this client |
| Webhook spoofing (outbound trust) | HMAC-SHA256 over `timestamp:payload` + nonce headers; secret stripped from payload | [call-webhook.job.ts#L54-L70](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L54-L70) | Secret exclusion via destructuring (L58) — a pattern to copy |
| Secrets leakage | Secret removed from webhook records for API responses | [remove-secret-from-webhook-record.ts](../../../packages/twenty-server/src/utils/remove-secret-from-webhook-record.ts) | Grep for similar scrubbing before returning any entity with credentials |
| DoS via query shape | Complexity cap + nested-1:N ban; body size limits; upload count/size limits | [common-base-query-runner.service.ts#L397-L431](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L397-L431), [main.ts#L66-L88](../../../packages/twenty-server/src/main.ts#L66-L88) | |
| API abuse | Token-bucket throttle (short+long) for API keys | [common-base-query-runner.service.ts#L350-L395](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L350-L395) | Browser-session limiting is elsewhere (captcha module exists; *investigate* coverage before relying on it) |
| XSS | React escaping by default; rich text via BlockNote | [modules/blocknote-editor](../../../packages/twenty-front/src/modules/blocknote-editor) | Audit any `dangerouslySetInnerHTML`/raw HTML rendering paths in rich text & email templates |
| CSRF | Bearer-token API (not cookie-auth for GraphQL) reduces classic CSRF; sessions exist for some flows | session setup [main.ts#L56](../../../packages/twenty-server/src/main.ts#L56) | *Investigate* which endpoints are cookie-authenticated before asserting safety |
| Info disclosure via errors | Standardized user-facing messages | [STANDARD_ERROR_MESSAGE](../../../packages/twenty-server/src/engine/api/common/common-query-runners/errors/standard-error-message.constant.ts) usage | Never put policy detail in denial errors |
| Dependency risk | Yarn resolutions pin transitive deps; patched packages vendored | [package.json#L27-L39](../../../package.json#L27-L39) | |
| Uploads | graphql-upload with max size/files | [main.ts#L74-L88](../../../packages/twenty-server/src/main.ts#L74-L88) | Check content-type validation in the file module before trusting |

## What a junior misses vs what a senior checks (security edition)

Junior: validates the happy-path input. Senior: asks who else can *reach* this code (API key? application? system job?), what the object's permission layer covers, whether new egress uses the secure client, whether errors leak, and what the *event* emitted by this write exposes to webhook consumers (payloads carry record data — a webhook to a compromised receiver is an exfil channel; field-level restrictions on webhook payloads: *investigate*).

## Pre-merge security checklist

- [ ] New endpoint/resolver: which authz layer covers it? Test for the denied case, not just the allowed one.
- [ ] Any raw SQL: identifiers never interpolated; values parameterized.
- [ ] Any outbound HTTP with user-influenced URL: goes through `SecureHttpClientService`.
- [ ] Any returned entity: secrets/tokens scrubbed.
- [ ] Any new event/webhook payload field: consider it public to tenant-configured receivers.
- [ ] Any `shouldBypassPermissionChecks: true`: one-line justification in the PR description.
- [ ] Errors: standardized message, typed code; no stack traces to clients.
- [ ] New queue job: decide and document swallow-vs-rethrow, timeout, and retry semantics explicitly.

Drill: take Kata 2 and Kata 4 from [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md) and run this checklist against them — every Blocking finding there maps to a row here. Interview cross-link: [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q7–Q9.
