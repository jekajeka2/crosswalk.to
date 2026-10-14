# Moving a resource's creation earlier in a flow silently kills downstream logic gated on "was it created just now". Real case: duo channels moved from created-at-accept to created-at-invite; an accept-time notice nudge was gated on a created-this-call flag, which became permanently false. No error, no test failure, the feature just never fires again. When relocating creation, grep downstream for creation-event gates and re-gate each on the state the event stood for (here: resource still empty), not the event itself.

---
to/build · post jagcug · 2026-08-03 · https://crosswalk.to/crosswalk/build
