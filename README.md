# Drosera DEX Volume Spike Trap (Hoodi Testnet)

This is a **unique alert-only trap** for **Drosera Network** on **Hoodi testnet**.  
The goal: **detect abnormal swap activity** (volume spike) on a DEX pair inside a short time window (~5 minutes).  
When spike happens, we **emit an alert**. We **do not** execute any on-chain action.

Live trap (Drosera app):  
**https://app.drosera.io/trap?trapId=0xf2d8d0de06c7c6f3208e1bdf41e68719ba55c094&chainId=560048**

---

## Why we built this

- In volatile moments (bot activity, sandwich, short-term pump), DEX pools see **fast changes**.  
- We want to **see it early** with conservative rules (avoid many false alarms).  
- This is **v1**: simple, reliable, and good for learning the Drosera lifecycle.

---

## What this trap does (high level)

- Watches a **Uniswap V2-like** pool (pair).  
- Uses a **5-minute window** (≈ 25 blocks on Hoodi) to avoid single-block noise.  
- Counts **trade-steps** (reserve change steps) and sums **base-token volume**.  
- If **count ≥ threshold** **AND** **volume ≥ threshold** → **Alert** (severity: high).  
- Has **cooldown** to stop alert spam.  
- **No response tx** (alert-only).

---

## Drosera lifecycle (how the contract runs)

The trap follows Drosera’s standard lifecycle:

1. **collect()**  
   - We take small snapshots from the watched pair over last ~25 blocks.  
   - We detect **reserve deltas** (small changes are ignored with `minTradeStep`).  
   - We compute **cumulated base volume** in window.

2. **shouldRespond()**  
   - Apply conservative thresholds:  
     - `txCountThreshold` (how many swaps/steps)  
     - `volumeThreshold` (required base volume)  
     - `minTradeStep` (ignore dust)  
   - If both pass → return `true` (means incident).

3. **shouldAlert()**  
   - We return **alert payload** with metadata (pool, window=5m, chain=hoodi).

> Note: We use a dummy `response_function` (like `helloworld(string)`) just to satisfy the Drosera plan schema. We **do not** act on-chain.

---

## Why it is unique (not just copy)

- Combines **multiple signals** (count + volume + min step) at the same time.  
- Uses **fixed block window** (~5 min) to reduce noise.  
- Has **cooldown** and **conservative** thresholds (good first step for real ops).  
- Designed for **alert-only**, making integration simple and safe.

---

## Key parameters

- `block_sample_size`: **25** blocks (≈ 5 minutes on Hoodi)  
- `cooldown_period_blocks`: **5**  
- `min_number_of_operators`: **1** (ok for test)  
- `private_trap`: **true** with **whitelist** (only our operators)  
- `response_function`: `helloworld(string)` (placeholder to keep Drosera pipeline valid)

Alert metadata:
- Title: `DEX TX Volume Spike Detected`  
- Severity: `high`  
- Labels: `{ pool, window="5m", chain="hoodi" }`

---

## What we did (short timeline)

1. **Idea & Design**  
   - Simple spike detection with **two conditions**: higher tx_count **and** higher volume.  
   - Ignore tiny steps using `minTradeStep`.  
   - Window = 25 blocks (~5m), **cooldown** = 5 blocks.

2. **Contract**  
   - Implemented Drosera Trap interface (no constructor params; compatible with trap registry).  
   - Pure alert-only design (no `respond` action).

3. **Testing**  
   - Wrote unit tests:  
     - Normal window → **no trigger**  
     - Spike window → **trigger**  
   - Local simulation with a **mock V2 pair** and reserve changes over time.

4. **Deployment** (Hoodi testnet)  
   - Deployed **mock pair** + **trap** for local validation.  
   - Then **updated existing trap** on Drosera explorer with our new artifact and plan.  
   - `drosera plan` + `drosera apply` to push changes (trap hash, window, cooldown, whitelist).

5. **Operators**  
   - Ran **Operator #1** and **Operator #2** on same server using Docker, with **different ports** to avoid conflicts (e.g., 31313/31314 and 31315/31316).  
   - Trap is **private**, so we **whitelisted** our operator addresses.  
   - Set **proof server URL** (`https://relay.hoodi.drosera.io`) so attestations work even if there are no pubsub peers.

6. **Liveness & sanity**  
   - Checked **liveness** from CLI and logs.  
   - Simulated reserve updates; normal pace → `ShouldRespond=false`; spike pattern → alert is ready.

---

## Risk profile (conservative)

- Good for early warning without too many false alarms.  
- Requires **both** high count **and** high volume.  
- Ignores **very small** movements.  
- Has **cooldown** to avoid spam during noisy bursts.

---

## Future ideas (v2 / v3)

- **Dynamic thresholds** (adaptive to pool liquidity).  
- Add **price impact** / slippage estimation.  
- Watch **multiple pools** and correlate signals.  
- Use **time-decay** weights inside the window (newer blocks count more).  
- Optional **response** path (e.g., call a notification contract or webhook relay).

---

## Links

- Live trap (Drosera app, Hoodi):  
  **https://app.drosera.io/trap?trapId=0xf2d8d0de06c7c6f3208e1bdf41e68719ba55c094&chainId=560048**

---

## Credits

- Built for **Drosera Network** on **Hoodi testnet**.  
- Thanks to the official Drosera contracts and examples for inspiration.  
- This repo focuses on the **alert logic** and **operator flow** (not heavy infra).
