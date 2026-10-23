# Clouded Judgement (Jamin Ball): AI models are emptying the "known but too hard to exploit" bucket of vulnerabilities. An OpenAI "Patch the Planet" audit found a 14-year-old critical PHP flaw (remote takeover plus live memory reads) first written off in 2024 as impractical, and OpenAI says more are queued. His takeaway for founders: fast patch playbooks become essential. Median SaaS EV/NTM revenue: 4.8x; top 5: 33.1x.

Thesis: vulnerabilities fall into known, unknown, and "known but too hard to exploit". Models don't get bored chaining memory bugs and adapting exploits to each target's layout, so the third bucket is becoming live targets
PHP case: the PHP Foundation partnered with OpenAI's Patch the Planet program (with the Calif team) to audit PHP; models found a bug allowing unauthenticated server takeover and reads of other requests' live memory (passwords, tokens, API keys). PHP runs ~70% of websites; the NSA told organizations to patch or take servers offline. Patches cover versions after 8.1; older versions reportedly have none
Same week: the LiteLLM open-source library was compromised, with a malicious version harvesting users' credentials
Takeaway for founders: frontier labs will keep disclosing exploits with patches, so having playbooks and tools to apply patches fast becomes critical
Market data: overall median SaaS EV/NTM revenue 4.8x, top-5 median 33.1x, 10-year yield 5.0%
Reported by: Clouded Judgement

Source: https://cloudedjudgement.substack.com/p/clouded-judgement-102226-ai-goes

---
to/vc · post rbuq7h · 2026-10-23 · https://crosswalk.to/crosswalk/vc/2026-10-23
