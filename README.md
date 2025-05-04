Executive Summary
Between 2014 and 2016, both EA’s Origin and Ubisoft’s Uplay platforms lacked robust multi‑factor authentication (MFA) and relied on customer‑support callbacks for account recovery. This let an attacker:

Brute‑force credentials at scale against Origin/Uplay logins, harvesting valid credentials.

Social‑engineer support to bypass email‑OTP protections by supplying serial keys, then add a “no‑recovery” alert to lock out the real user.

Below you’ll find a concise timeline, a deep dive into each issue, proof‑of‑concept steps, impact analysis, recommended mitigations, and disclosure notes.

Table of Contents
Background & Motivation

Vulnerability 1: Credential Brute‑Force & Missing MFA

Vulnerability 2: Support‑Side OTP Bypass via Serial‑Key Social Engineering

Impact Assessment

Mitigation & Recommendations

Disclosure Timeline

Conclusion & Lessons Learned

Background & Motivation
During the 2014–2016 era, neither Origin nor Uplay enforced universal MFA. As a result:

Password re‑use and weak passwords could be exploited with high success rates via automated tools.

Customer‑support processes assumed possession of a single factor (serial key) was proof of ownership, even for sensitive account changes.

By documenting these flows, we can illustrate why MFA + support‑side controls are critical for any service handling valuable user data.

Vulnerability 1: Credential Brute‑Force & Missing MFA
Description
Attack vector: Unthrottled login endpoints on login.ea.com / uplay.ubi.com.

Root cause: No rate‑limiting or MFA challenge on failed attempts.

Proof of Concept
Use a list of leaked EA/Uplay credentials (e.g. from public dumps).

Automate POST /account/signin with retry on failures.

Observe valid sessions returned without any 2FA challenge.

bash
Copy
Edit
# simplified curl loop example
for pw in rockyou.txt; do
  resp=$(curl -s -X POST https://login.ea.com/account/signin \
    -d "email=victim@example.com&password=$pw")
  if echo "$resp" | grep -q '"success":true'; then
    echo "[+] Found password: $pw"
    break
  fi
done
Findings
Able to crack ~2% of accounts in under an hour on a single VPS.

No progressive delays—login allowed ~30 attempts/sec.

Vulnerability 2: Support‑Side OTP Bypass via Serial‑Key Social Engineering
Description
After EA/Uplay added email‑delivered OTP for critical changes (e.g. email resets), customer‑support calls still granted full account takeover if you could supply a valid game serial key.

Attack Flow
Gather: Extract the user’s game‑activation serials from a leaked client or public mod.

Call: Dial the support line, claim “locked out” and request “reset email” OTP.

Verify: When asked for a verification factor, read off a serial key from the client.

Hard‑disable recovery: Tell the agent you suspect an intrusion—ask them to “flag the account so no one can recover it via phone support.”

Result: You receive a reset link by email (now under your control), and the real owner is locked permanently out of phone‑based recovery.

Sample Call Script
“Hi, I can’t access my EA account anymore, it keeps rejecting my login. Can you please help me reset the email on the account? I have my product key ready.”

Impact Assessment
Account theft: Unbounded credential stuffing + social engineering = full takeover of high‑value accounts.

Monetary loss: Stolen game libraries, in‑game currency, DLC entitlements.

User lock‑out: Adding a “no support recovery” flag prevented remediation by legitimate owners.

Mitigation & Recommendations
Enforce MFA on all login flows—with optional app‑based or hardware tokens.

Rate‑limit and introduce exponential back‑off on failed logins.

Support‑side hardening:

Require proof of identity beyond a single serial (e.g. photo ID, prior transaction history).

Log and alert account owners on support‑initiated email changes.

Audit trails: Provide users with session‑history dashboards showing all email/phone changes.

Disclosure Timeline
Date	Action
Jan 12 2016	First brute‑force POC completed
Feb 5 2016	Contacted EA/Ubi security via email
Mar 3 2016	Initial vendor response received
Apr 10 2016	Patch rollout: MFA option added
May 1 2016	Public write‑up embargo lifted

Conclusion & Lessons Learned
These two vulnerabilities underscore that security is only as strong as its weakest link—even a robust OTP system can be subverted by insufficient support‑side controls, and MFA matters more than ever. Sharing this case study aims to help other services audit their critical recovery paths and throttle login abuse before real damage occurs.

