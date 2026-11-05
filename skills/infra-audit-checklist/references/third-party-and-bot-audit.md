---
name: third-party-and-bot-audit
description: "External-host enumeration, privacy manifest fields and the five-layer deployed-bot audit, with the commands and report templates."
---

# Third-party software and deployed-bot audit

## Step 1 — pick the analysis surface

| Software type | Source available | Approach |
|---|---|---|
| Open source | repository exists | Clone and read the code. |
| Closed source mobile | store binary only | Static analysis of metadata only. State the limitation. |
| Closed source desktop | binary distribution | Inspect strings and runtime network calls. State the limitation. |
| Browser extension | packaged extension | Read the manifest, then grep the minified script. |

Announce the depth in the first line of the reply.
## Step 2 — enumerate external hosts (the leading signal)
Produce the external-host list before reading anything else.

```bash
# Hardcoded URLs in the app target (exclude tests, comments, placeholders)
grep -rnE "https?://[a-zA-Z0-9.-]+" <repo>/<app-target>/ --include="*.swift" --include="*.kt" \
  | grep -v "^\s*//" | grep -vE "(\.example|localhost|127\.0\.0\.1|192\.168|10\.|placeholder|invalid|\.local|\.ts\.net|\.test)" \
  | grep -oE "https?://[a-zA-Z0-9.-]+(/[^\"' ]*)?" | sort -u

# SwiftPM dependencies (transitive SDKs are the surprise)
grep -E "XCRemoteSwiftPackageReference|repositoryURL" <repo>/<App>.xcodeproj/project.pbxproj
# Android dependencies
grep -E "implementation |api " <repo>/app/build.gradle* <repo>/gradle/libs.versions.toml 2>/dev/null
# Cross-platform
grep -E '"[^"]*":' <repo>/package.json <repo>/pubspec.yaml 2>/dev/null
```

Classify each hit. **User-controlled server**: expected, that is the product. **Author's own cloud**: acceptable if the project is open and auditable, suspicious if closed. **Third-party relay, CDN or analytics**: red flag — the recurring offenders are Mixpanel, Sentry, Firebase, Crashlytics, AppsFlyer, Amplitude, PostHog, Datadog, Adjust, Segment, Google Analytics and Matomo; present and not user-toggleable means the verdict is no.

Not every SDK appears in the dependency manifest — many link through CocoaPods, Carthage or a vendored framework, so unpack the bundle and grep the binary too:

```bash
otool -L App.app/Frameworks/*.framework/*   # linked dylibs
strings App.app/Binary | grep -iE "(mixpanel|sentry|firebase)" | head
```

## Step 3 — the privacy manifest, field by field
The manifest is the developer's claim. The decisive fields:

- `NSPrivacyTracking` — true means the tracking-consent dialog is mandatory and the developer admits to tracking. False while an analytics SDK sits in the binary means the manifest is wrong.
- `NSPrivacyTrackingDomains` — empty means clean; any entry is a red flag.
- `NSPrivacyCollectedDataTypes` — each entry carries what is collected, whether it is linked to identity, and the purposes. `AppFunctionality` alone is normal; `Analytics` or `ThirdPartyAdvertising` is a smell; plural purposes combined with linked identity is a red flag.
- `NSPrivacyAccessedAPITypes` — the platform API categories the app claims to access, each with a reason code. Compare against what the code actually calls.
## Step 4 — transport security

| Configuration | Meaning | Verdict |
|---|---|---|
| absent | HTTPS enforced, plain HTTP denied | Best |
| local networking allowed | plain HTTP to local-network hosts only | Fine for a self-hosted client |
| arbitrary loads allowed | plain HTTP to any host | Red flag |
| exception domains | per-domain exemptions | Investigate each one |
## Step 5 — logging and credential storage

```bash
grep -rnE "(Logger\(|os\.log|print\()" <repo>/<app-target>/ \
  | grep -iE "(token|key|password|secret|cookie|auth|url|request)" | head -20
grep -rnE "kSec|Security\.SecItemAdd" <repo>/<app-target>/                          # keychain: good
grep -rnE "UserDefaults\.standard\.set.*(token|password|secret)" <repo>/<app-target>/  # bad
```

Confirm the build guard before flagging debug-only logging — it does not ship in release builds.
## Step 6 — network behaviour
For open source: every request construction (endpoint, method, headers, body). The request body is the payload — read each encoded structure for the fields sent upstream, especially device tokens, install keys and user identifiers. Check background sessions, which run while the app is suspended. For closed source: static analysis only, and state the limits.

## The five layers of a deployed-bot audit
### 1. Real usage

```bash
sqlite3 <db> "SELECT COUNT(*) FROM messages"
sqlite3 <db> "SELECT COUNT(DISTINCT chat_id) FROM messages"
```

A bot with a handful of messages has no evidence behind its architecture. Say so in the report.

### 2. Response reliability

```bash
for i in $(seq 1 6); do node probe-real-prompt.mjs; done   # prints chars and ms
```

Report the empty rate and the latency (mean and max) with numbers. Usual root cause: the output-token cap is too small for the model's reasoning, so the budget is consumed internally and the answer arrives empty. Confirm it in the completion-versus-reasoning token counts.

### 3. What the user actually sees
Reasoning leak ratio `reasoning chars / total chars` (above 50% is critical); chunking with no empty messages and no orphaned chunks; escaping where a single quote in place of a double quote breaks the whole render.

### 4. Coherence between entry paths

```bash
grep -rn "buildMessages\|persona\|retrieveKnowledge" src/ | sort
```

Each path can carry its own persona, corpus and rules. Different corpora across paths is a first-class finding.

### 5. Domain coverage

```bash
grep -o '"titulo":[^,]*' syllabus.json | sed 's/.*: *"\(.*\)"/\1/' > /tmp/topics.txt
while read -r t; do grep -qiF "$t" corpus.md || echo "MISSING: $t"; done < /tmp/topics.txt
```

Zero of N means the bot answers from general knowledge, cannot cite its source, and cannot report which topics it failed. Also verify what the corpus claims is unavailable — if it is in fact published, half the syllabus is excluded.

## Verifying a finding against a primary source
Download the official document, extract the count and the enumeration, then compare **code, corpus and source** against each other. The discrepancy between code and corpus is where the real bug lives.

## Report templates
Third-party software: **verdict** (one line, yes or no, with the reason); **data-flow table** (each external host, what flows each way, user-configurable yes or no); **code evidence** (file and line); **defaults to change** (only when the defaults are not already strict); **self-hosting path** (how to replace the third-party relay or CDN with user-controlled infrastructure).

Deployed bot: **verdict** (one line — does it serve the user today, and why); **critical findings** (what happens, the measured number, the root cause, the concrete fix); **medium findings** in a table; **what is working** (part of the report — it tells the operator what not to touch); **direct answers** to each question asked; **phased fix plan** (first what makes the bot respond, then content, then features).

Verdict categories for third-party software: yes / yes with one tweak / maybe (closed source, known-trustworthy author) / no (tracking SDK, plaintext credentials, arbitrary plain HTTP, manifest contradicting the binary).
