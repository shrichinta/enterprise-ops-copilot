Screenshots from testing on 25 Nov 2026.

demo/            portfolio / interview evidence (v1 + v2 in Colab, Gemini)
  01 header        model + RBAC role switch
  02-03 INV-5003   +18.00% fuel surcharge flagged, tool trace, citation
  04 INV-5001      clean invoice, VAT 5% correctly not a false alarm
  05 INV-5011      idempotent write-back (ALREADY_FLAGGED, same log ID)
  06 INV-5011      prompt injection detected and ignored, mismatch still flagged
  07-08 margin     analyst: restricted / manager: 24% (RBAC enforced in the tool)
  transcript       full text of the INV-5003, INV-5001, INV-5014 runs

troubleshooting/ real issues hit and how each was fixed (good "lessons learned" story)
  t0  eval run with an OpenAI key in the Gemini slot -> false passes -> eval now aborts on model errors
  t1  503 high demand        -> retry + backoff, pinned secondary model, degraded mode
  t2  404 model closed to new keys -> never hard-code a model ID; pin via config after checking availability
  t3  429 free-tier quota    -> paid/Vertex endpoint in production; circuit breaker
  t4  model availability check (which models answered)
  t5  v1 margin answer used the wrong tool -> prompt + tool description fixed in v2
