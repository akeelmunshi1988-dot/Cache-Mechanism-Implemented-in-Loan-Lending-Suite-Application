# Redis Caching in a Loan Lending Suite

> A concrete placement guide for using Redis caching, rate limiting, and distributed locking in a loan-lending platform.


## Overview

Redis is most valuable in a lending platform when it reduces expensive or repetitive reads without becoming the system of record for financially authoritative data.

This guide focuses on three Redis patterns:

- **Cache-aside caching** for expensive or frequently read data.
- **Atomic counters** for distributed API rate limiting.
- **Distributed locks** for coordinating high-stakes operations across application instances.

> **Core principle:** Cache data that is expensive to fetch or compute and does not need to be perfectly fresh. For data that must be authoritative at the moment of a financial decision, use Redis for coordination rather than as the source of truth.

## Contents

1. [Credit Bureau / Credit Score Lookups](#1-credit-bureau--credit-score-lookups)
2. [Interest Rate Cards / Loan Product Configuration](#2-interest-rate-cards--loan-product-configuration)
3. [EMI / Loan Calculators](#3-emi--loan-calculators-for-high-traffic-public-pages)
4. [Loan Application Status / Dashboard Views](#4-loan-application-status--dashboard-views)
5. [Underwriting Rules / Decision Engine Configuration](#5-underwriting-rules--decision-engine-configuration)
6. [KYC / Document Verification Status](#6-kyc--document-verification-status)
7. [Rate Limiting External APIs](#7-rate-limiting-external-credit-bureau--verification-api-calls)
8. [Distributed Locking for Disbursement](#8-distributed-locking-for-loan-disbursement--approval-actions)
9. [What You Should NOT Cache](#what-you-should-not-cache-in-this-domain)
10. [Summary](#summary-table-for-this-domain)

Applying the general patterns from before to a loan lending domain specifically — here's where each fits, with the reasoning for *why* that particular piece is a good (or bad) caching candidate.

### 1. Credit Bureau / Credit Score Lookups (High-Value Target)

This is usually the **single best candidate** in the entire system. Credit bureau APIs (Experian, Equifax, CIBIL, etc.) are:

- **Slow** (external network call, often 1-5+ seconds)
- **Rate-limited** and often **billed per call** — every avoidable call has a direct dollar cost
- Reasonably **stable within a short window** — a credit score doesn't change meaningfully within, say, 24 hours

```csharp
public async Task<CreditReport> GetCreditReportAsync(string ssn)
{
    var cacheKey = $"credit-report:{HashSsn(ssn)}";  // never cache/log raw SSN as the key — hash it
    var cached = await _cache.GetStringAsync(cacheKey);
    if (cached != null)
        return JsonSerializer.Deserialize<CreditReport>(cached)!;

    var report = await _creditBureauClient.PullReportAsync(ssn);   // expensive, billed external call

    await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(report),
        new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24) });

    return report;
}
```

**Important compliance note**: many jurisdictions (FCRA in the US, for example) have rules about how long credit report data can be retained/reused before a fresh pull is legally required for a new decision — the TTL here isn't just a performance knob, it needs to align with actual regulatory permissible-use windows. This is worth confirming with compliance/legal, not just picking an arbitrary number.

### 2. Interest Rate Cards / Loan Product Configuration

Rate sheets, product terms, fee schedules — this is classic **reference data**: read constantly (every quote, every calculator use), written rarely (maybe daily or on a scheduled rate change).

```csharp
public async Task<List<RateCardEntry>> GetActiveRateCardAsync(string productType)
{
    var cacheKey = $"rate-card:{productType}";
    var cached = await _cache.GetStringAsync(cacheKey);
    if (cached != null)
        return JsonSerializer.Deserialize<List<RateCardEntry>>(cached)!;

    var rates = await _repository.GetActiveRatesAsync(productType);
    await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(rates),
        new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1) });
    return rates;
}

// Explicitly invalidate the moment rates actually change — don't just wait for TTL
public async Task UpdateRateCardAsync(string productType, List<RateCardEntry> newRates)
{
    await _repository.SaveRatesAsync(productType, newRates);
    await _cache.RemoveAsync($"rate-card:{productType}");   // force next read to pick up the new rates immediately
}
```

This is exactly the "read-heavy, write-light" profile flagged as ideal in the general answer — every loan calculator hit, every quote generation, every underwriting check reads this.

### 3. EMI / Loan Calculators for High-Traffic Public Pages

If there's a public-facing "estimate your monthly payment" calculator (very common on lending sites), the underlying rate/fee/term data it pulls should be cached — the calculation itself is cheap CPU work, but the **inputs** (rate card, fee schedule) shouldn't hit the DB on every keystroke/slider-drag on a marketing page.

```csharp
public async Task<EmiEstimate> CalculateEmiAsync(decimal principal, int termMonths, string productType)
{
    var rates = await GetActiveRateCardAsync(productType);   // cached, per above
    return _emiCalculator.Calculate(principal, termMonths, rates);   // cheap, in-memory computation, no need to cache the result itself
}
```

### 4. Loan Application Status / Dashboard Views

If borrowers or loan officers repeatedly poll or refresh a status/dashboard page ("where is my application?"), cache the aggregated view for a short window rather than re-running potentially complex joins/aggregations on every refresh.

```csharp
public async Task<LoanApplicationSummary> GetApplicationStatusAsync(string applicationId)
{
    var cacheKey = $"loan-status:{applicationId}";
    var cached = await _cache.GetStringAsync(cacheKey);
    if (cached != null)
        return JsonSerializer.Deserialize<LoanApplicationSummary>(cached)!;

    var summary = await _repository.BuildStatusSummaryAsync(applicationId);   // possibly several joined tables

    await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(summary),
        new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(30) });
    return summary;
}

// CRITICAL: invalidate immediately on any status-changing event, don't rely purely on the short TTL
public async Task UpdateApplicationStageAsync(string applicationId, ApplicationStage newStage)
{
    await _repository.UpdateStageAsync(applicationId, newStage);
    await _cache.RemoveAsync($"loan-status:{applicationId}");
}
```

**Short TTL is deliberate here** — unlike rate cards, application status is something the borrower cares about being accurate *right now*, especially right after they take an action (uploaded a document, e-signed something). A 30-second-ish TTL plus explicit invalidation on write gives good read performance without meaningfully stale "where's my loan" answers.

### 5. Underwriting Rules / Decision Engine Configuration

If underwriting decisions are driven by a configurable rules engine (DTI thresholds, LTV limits, approval matrices), that configuration is read on **every single application evaluated** but changed infrequently (maybe by risk/compliance teams periodically) — another strong reference-data caching candidate, same pattern as the rate card.

```csharp
public async Task<UnderwritingRuleSet> GetActiveRuleSetAsync(string productType)
{
    var cacheKey = $"underwriting-rules:{productType}";
    // same cache-aside pattern as rate card above
}
```

### 6. KYC / Document Verification Status

Document/identity verification often involves calling a **third-party verification service** (similar cost/latency profile to credit bureau calls) — cache the verification *result* for a bounded period so re-checking the same application's status page doesn't re-trigger external calls.

```csharp
public async Task<KycStatus> GetKycStatusAsync(string customerId)
{
    var cacheKey = $"kyc-status:{customerId}";
    // cache-aside — but keep TTL short-ish (e.g., 15 min) and always invalidate on any
    // webhook/callback from the verification provider signaling a status change
}
```

### 7. Rate Limiting External Credit Bureau / Verification API Calls

Directly reusing the rate-limiting pattern from the general answer — credit bureau contracts often have **per-second/per-day call caps**, and exceeding them can mean extra charges or service suspension. Redis's atomic counters are ideal for enforcing this across all your app instances.

```csharp
public async Task<bool> CanCallCreditBureauAsync()
{
    var key = "ratelimit:credit-bureau:global";
    var count = await _redis.StringIncrementAsync(key);
    if (count == 1)
        await _redis.KeyExpireAsync(key, TimeSpan.FromSeconds(1));
    return count <= _maxCallsPerSecond;
}
```

### 8. Distributed Locking for Loan Disbursement / Approval Actions

This is a genuinely important one in lending specifically — you need to guarantee a loan is **never disbursed twice**, and if the approval/disbursement workflow can be triggered from multiple app instances (or retried by a queue/job processor), a distributed lock prevents a duplicate payout.

```csharp
public async Task DisburseLoanAsync(string applicationId)
{
    var lockKey = $"lock:disburse:{applicationId}";
    var acquired = await _redis.StringSetAsync(lockKey, Environment.MachineName, TimeSpan.FromMinutes(2), When.NotExists);

    if (!acquired)
        throw new InvalidOperationException("Disbursement already in progress for this application.");

    try
    {
        if (await _repository.IsAlreadyDisbursedAsync(applicationId))
            return;   // idempotency check even with the lock, as defense-in-depth

        await _disbursementService.SendFundsAsync(applicationId);
        await _repository.MarkDisbursedAsync(applicationId);
    }
    finally
    {
        await _redis.KeyDeleteAsync(lockKey);
    }
}
```

This is a much higher-stakes use of Redis than the read-through caching above — a bug here means real money moving twice, so treat this piece with the same care as any financial-transaction code (idempotency checks as defense-in-depth, not just the lock alone).

### What You Should NOT Cache in This Domain

| Data | Why not |
|---|---|
| **Real-time account balances / outstanding loan balance** | Needs to be authoritative and current at the moment of any financial decision — caching risks acting on stale numbers. |
| **Final underwriting decision for a specific application** | This is a point-in-time legal/compliance decision tied to the exact data available at that moment — don't reuse a cached "approved" result if inputs (income, credit) have since changed. |
| **Compliance/audit trail data** | Needs to reflect the true, current state of record for regulatory purposes — never served from a cache that could be stale during an audit. |
| **PII in plaintext within cache keys or values** | If caching anything involving SSN, account numbers, etc., encrypt/hash appropriately and set short TTLs — Redis is an additional place sensitive data now lives, which has real security/compliance implications (PCI DSS, GLBA, etc. depending on jurisdiction). |
| **In-flight application data mid-edit** | If a borrower is actively filling out a multi-step application, that data usually needs to be durably persisted (DB), not just cached — a cache eviction shouldn't be able to lose someone's half-completed loan application. |

### Summary Table for This Domain

| Component | Cache? | TTL guidance | Why |
|---|---|---|---|
| Credit bureau report | ✅ Yes | Hours (bounded by regulatory reuse window) | Expensive, billed, rate-limited external call |
| Interest rate card / product config | ✅ Yes | Hours + explicit invalidation on change | Read constantly, changes rarely |
| Underwriting rule set | ✅ Yes | Hours + explicit invalidation on change | Same profile as rate card |
| EMI calculator inputs (rates/fees) | ✅ Yes (via #2) | Same as rate card | High-traffic public calculator |
| Loan application status/dashboard | ✅ Yes, short TTL | Seconds + invalidate on any status change | Needs to feel "live" without hammering the DB on every poll |
| KYC/verification status | ✅ Yes, short TTL | Minutes | Avoids re-hitting external verification provider |
| Credit bureau/verification API rate limiting | ✅ Yes (counters, not data caching) | Per rate-limit window | Protects against exceeding paid API quotas |
| Loan disbursement action | ✅ Distributed lock (not caching, but same Redis infrastructure) | Minutes | Prevents duplicate fund disbursement across instances |
| Final underwriting decisions, balances, audit data | ❌ No | — | Needs to be authoritative/current for financial and compliance correctness |

**The organizing principle carried over from the general answer**: cache things that are **expensive to compute/fetch and don't need to be perfectly fresh** (rate cards, credit reports, rules config) aggressively; treat things that are **cheap to fetch but must be authoritative right now** (balances, final decisions, disbursement state) as things Redis should coordinate (locks, rate limits) rather than cache as data
