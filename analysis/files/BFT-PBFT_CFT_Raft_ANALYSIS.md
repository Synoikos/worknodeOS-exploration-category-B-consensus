# File Analysis: BFT-PBFT_CFT_Raft.MD

**File**: `source-docs/BFT-PBFT_CFT_Raft.MD`
**Category**: B - Consensus & BFT
**Analysis Date**: 2025-11-20
**Analyst**: Claude (Sonnet 4.5)

---

## 1. Executive Summary (3-5 sentences)

This document presents a comprehensive Q&A-style discussion on Byzantine Fault Tolerance (BFT) vs. Crash Fault Tolerance (CFT) in the context of the Worknode distributed system. The core debate centers on whether Raft (CFT) is sufficient or if full BFT algorithms (PBFT, Tendermint) are needed, given that Raft cannot tolerate Byzantine (malicious) nodes. The document reveals that the current system uses a **hybrid approach**: Raft for consensus coordination combined with cryptographic defenses (Ed25519 signatures, Merkle proofs, capability security) that provide Byzantine resistance at the data integrity level. The key recommendation is to **NOT implement full PBFT in v1.0**, but instead add three lightweight mitigations: (1) audit trails for Byzantine leader detection, (2) client-side quorum verification, and (3) leader term limits—achieving 95% Byzantine resistance with minimal complexity overhead (20-30 hours vs. 100+ hours for PBFT).

---

## 2. Architectural Alignment

### Fits Worknode Abstraction?
**YES** - The document demonstrates deep understanding of the Worknode architecture and proposes solutions that align with its fractal, capability-secure design.

**Evidence**:
- Recognizes the system has "Raft + cryptographic verification layers" (not pure Raft)
- References existing components: `include/consensus/raft.h`, `include/algorithms/merkle.h`, `include/security/capability.h`
- Proposes mitigations that leverage existing capability and Merkle proof systems
- Maintains NASA Power of Ten compliance (all proposed solutions use bounded iteration)

### Impact on Capability Security?
**Minor Enhancement** - Proposes strengthening existing capability security with additional verification layers.

**Details**:
- Audit trail proposal uses existing Ed25519 signatures
- Client quorum verification builds on existing multi-signature capabilities
- No breaking changes to capability model
- Extends security without adding new primitives

### Impact on Consistency Model?
**None/Minor** - Works within existing layered consistency (LOCAL/EVENTUAL/STRONG).

**Details**:
- Raft remains the strong consistency mechanism (unchanged)
- Proposed mitigations add verification, not new consistency levels
- Leader term limits affect availability slightly (60-second re-elections) but not consistency semantics
- CRDT eventual consistency layer unaffected

---

## 3. Criterion 1: NASA Compliance Status

**Status**: **SAFE** - All proposed solutions maintain NASA Power of Ten compliance.

### Analysis by Proposal:

**Option 1: Audit Trail (SAFE)**
- Uses bounded log storage (existing Raft log, append-only)
- Cross-follower comparison: O(n²) where n = follower count (bounded by MAX_REPLICAS)
- No recursion, no dynamic allocation
- Complexity: ~8 (within NASA limit of 15)

**Option 2: Client Quorum Verification (SAFE)**
- Client queries bounded set of followers (≤ MAX_REPLICAS)
- Hash comparison: O(n) bounded iteration
- No unbounded loops or recursion
- Complexity: ~6

**Option 3: Leader Term Limits (SAFE)**
- Simple timer check (constant time)
- No new loops, no recursion
- Complexity: ~3

**Deferred Option: Full PBFT (REVIEW)**
- PBFT requires 3-phase protocol (pre-prepare, prepare, commit)
- Additional message rounds increase code paths
- Would need careful complexity analysis
- Estimated complexity: 12-15 (at NASA limit)

**Verdict**: v1.0 mitigations are NASA-safe. Full PBFT would require compliance re-certification.

---

## 4. Criterion 2: v1.0 vs v2.0 Timing

**Classification**: **v1.0 ENHANCEMENT** (not CRITICAL, but recommended before production)

### Rationale:

**NOT v1.0 CRITICAL because**:
- Wave 4 RPC layer not blocked by this
- System functions correctly with Raft CFT for trusted internal deployments
- Byzantine leader attacks require insider compromise (low probability in enterprise setting)

**IS v1.0 ENHANCEMENT because**:
- Adds production-grade resilience with minimal cost (20-30 hours)
- Enterprise deployments expect audit trails
- Client verification is industry best practice
- 95% Byzantine resistance vs. 0% with pure Raft

**v2.0+ Deferral**:
- Full PBFT (100+ hours, 3-5× latency overhead)
- Multi-signature scheme (medium complexity)
- Advanced monitoring/auto-recovery

**Recommended Timing**:
- Implement 3 lightweight mitigations in **Phase 8 (System Security)** - BEFORE v1.0 launch
- Defer full PBFT to v2.0+ unless customer requirement surfaces

---

## 5. Criterion 3: Integration Complexity

**Score**: **4/10** (MEDIUM)

### Breakdown:

**Audit Trail**: **2/10** (LOW)
- **Effort**: 10-15 hours
- **Changes**:
  - Extend existing Raft log structure (add `AppendEntriesRPC` logging)
  - Implement `detect_byzantine_leader_post_round()` (~50 lines)
  - Add periodic comparison job (timer-based)
- **Risks**: Minimal - builds on existing Raft infrastructure
- **Testing**: 3-5 unit tests

**Client Quorum Verification**: **5/10** (MEDIUM)
- **Effort**: 8-12 hours
- **Changes**:
  - Modify client RPC layer to query multiple followers
  - Implement hash comparison logic (~80 lines)
  - Add timeout/retry handling
- **Risks**: Client-side complexity, potential latency increase
- **Testing**: 5-8 integration tests (normal + Byzantine scenarios)

**Leader Term Limits**: **2/10** (LOW)
- **Effort**: 2-3 hours
- **Changes**:
  - Add `LEADER_TERM_MAX_DURATION_MS` constant
  - Implement `enforce_leader_term_limits()` (~20 lines)
  - Trigger voluntary step-down
- **Risks**: Minimal - simple timer logic
- **Testing**: 2 unit tests

**Full PBFT (if chosen)**: **9/10** (EXTREME)
- **Effort**: 100+ hours
- **Changes**:
  - Implement 3-phase PBFT protocol
  - Replace Raft election/replication logic
  - Re-architect message flows
  - Increase message overhead (3-5×)
- **Risks**: High - fundamental redesign, re-certification needed
- **Testing**: 30+ tests, formal verification recommended

**Justification for Medium (4/10) Score**:
The recommended v1.0 approach (3 mitigations) averages to ~4/10 complexity:
- (2 + 5 + 2) / 3 = 3, but accounting for integration testing and edge cases → **4/10**

---

## 6. Criterion 4: Mathematical/Theoretical Rigor

**Classification**: **RIGOROUS** (strong theory, production-proven elsewhere)

### Evidence:

**Raft CFT (PROVEN)**:
- Formal TLA+ specification exists ([Raft paper](https://raft.github.io/))
- Proven correctness: Safety, liveness, leader election
- Production use: etcd, Consul, Kubernetes (10+ years)
- **Confidence**: 99%

**BFT Theory (PROVEN)**:
- PBFT: Formal proof of Byzantine tolerance (n ≥ 3f + 1)
- Tendermint: Refinement of PBFT with liveness proofs
- Production use: Hyperledger Fabric, Cosmos blockchain
- **Confidence**: 95%

**Cryptographic Defenses (PROVEN)**:
- Ed25519: Formally verified signature scheme
- Merkle proofs: Information-theoretic security
- Bloom filters: Probabilistic correctness (false positive rate calculable)
- **Confidence**: 99%

**Proposed Mitigations (RIGOROUS)**:
- Audit trail: Standard technique (no novel algorithm)
- Client quorum verification: Adapted from blockchain light clients
- Term limits: Common practice in Raft implementations (etcd uses this)
- **Confidence**: 85% (needs implementation validation)

**Gaps/Weaknesses**:
- No formal proof provided for hybrid Raft+Crypto = Byzantine resistance
- Audit trail detection is post-facto (reactive, not preventive)
- Client verification adds latency (no performance analysis provided)

**Overall**: Solid theoretical foundation, but hybrid approach needs empirical validation.

---

## 7. Criterion 5: Security/Safety Implications

**Classification**: **SECURITY-CRITICAL**

### Security Aspects:

**Threat Model Clarification (CRITICAL)**:
- Document correctly identifies that Byzantine leader in pure Raft = safety violation
- Current hybrid system: Merkle proofs prevent *data* corruption, but leader can still cause split-brain
- Mitigation strategy: Accept limited Byzantine leader damage, detect quickly, minimize blast radius

**Security Benefits**:
1. **Audit Trail**: Byzantine leader detection within 1 consensus round (post-facto)
2. **Client Verification**: Prevents client-side impact of split-brain
3. **Term Limits**: Limits Byzantine leader damage to 60 seconds max

**Residual Risks**:
1. **Byzantine Leader Window**: Between becoming leader and detection (~60 seconds)
   - **Impact**: Temporary inconsistency, possible data loss for in-flight requests
   - **Mitigation**: Client retries + quorum verification
2. **Nonce Cache Overflow**: LRU cache (10,000 entries) can evict old nonces
   - **Impact**: Replay attack window after 10,000 subsequent operations
   - **Mitigation**: Nonces also expire with capability expiry
3. **Collusion Attack**: 2+ malicious admins can bypass audit trail
   - **Impact**: Sustained Byzantine behavior if majority of approvers compromised
   - **Mitigation**: Multi-party code signing (separate concern, addressed in another doc)

**Safety Aspects (NASA)**:
- All mitigations maintain bounded execution
- No new dynamic allocation
- Failure modes well-defined (degradation to read-only vs. crash)

**Severity**: HIGH (Byzantine leader can cause temporary data loss), but **mitigated to MEDIUM** with proposed defenses.

---

## 8. Criterion 6: Resource/Cost Impact

**Classification**: **LOW-COST** (< 1% overhead for v1.0 mitigations)

### Cost Breakdown:

**Audit Trail**:
- **Storage**: Raft log already exists; adds ~64 bytes per AppendEntries RPC (signature metadata)
  - For 1000 ops/sec: 64 KB/sec = 5.5 GB/day (manageable with log rotation)
- **CPU**: Cross-follower hash comparison runs periodically (e.g., every 10 seconds)
  - Complexity: O(n²) where n = follower count (≤10), ~100 hash comparisons
  - **Estimate**: < 0.1% CPU
- **Network**: No additional network traffic (uses existing Raft log)

**Client Quorum Verification**:
- **Latency**: Client queries 3-5 followers instead of leader only
  - Additional RTT: ~2ms per follower (parallel queries)
  - **Total added latency**: ~5-10ms (acceptable for most workloads)
- **CPU**: Hash comparison per query (negligible, ~1μs)
- **Network**: 2-4× bandwidth increase (queries multiple nodes)
  - For 1000 req/sec: 1000 req × 4 nodes = 4000 requests/sec total
  - **Estimate**: Manageable with modern NICs (10 Gbps+)

**Leader Term Limits**:
- **CPU**: Timer check per heartbeat (~0.001% overhead)
- **Latency**: Forced re-election every 60 seconds
  - Raft election: ~100-200ms downtime
  - **Impact**: 0.2-0.4% availability reduction (99.996% → 99.6%)

**Full PBFT (if implemented)**:
- **Latency**: 3-5× increase (3-phase protocol)
- **Throughput**: 40-60% reduction (more message rounds)
- **Network**: 3-5× message overhead
- **Cost**: HIGH (> 5% overhead)

**Overall v1.0 Impact**: **< 1% CPU, ~10ms latency, 2-4× network** = **LOW-COST**

---

## 9. Criterion 7: Production Deployment Viability

**Classification**: **PROTOTYPE-READY** (3-6 months validation for v1.0 mitigations)

### Readiness Assessment:

**Audit Trail**:
- **Maturity**: Technique proven in blockchain full nodes (Ethereum, Bitcoin)
- **Implementation Risk**: LOW (straightforward log comparison)
- **Validation Needed**:
  - Confirm detection latency acceptable (< 10 seconds target)
  - Test with simulated Byzantine leader (inject split-brain)
  - Verify log storage scaling (rotation policies)
- **Timeline**: 1-2 months testing

**Client Quorum Verification**:
- **Maturity**: Standard in Byzantine-tolerant systems (blockchain SPV wallets)
- **Implementation Risk**: MEDIUM (client-side complexity, error handling)
- **Validation Needed**:
  - Latency impact measurement (target: < 10ms P95)
  - Retry logic testing (network failures, follower crashes)
  - Load testing (ensure 4× traffic sustainable)
- **Timeline**: 2-3 months testing

**Leader Term Limits**:
- **Maturity**: Used in etcd production deployments
- **Implementation Risk**: LOW (simple timer)
- **Validation Needed**:
  - Confirm 60-second limit doesn't cause excessive elections
  - Test under load (ensure re-elections don't cascade)
- **Timeline**: 1 month testing

**Integration Testing**:
- Byzantine leader + audit trail (detection time)
- Byzantine leader + client verification (client unaffected)
- Byzantine leader + term limit (damage contained)
- **Timeline**: 2 months

**Production Rollout**:
- Gradual rollout: Enable mitigations one at a time
- Monitoring: Add metrics for Byzantine detection events
- Rollback plan: Disable features via feature flags
- **Timeline**: 1-2 months

**Total Validation**: **3-6 months** from code complete to production-ready.

**Full PBFT**: Would require **12-18 months** (formal verification, extensive testing).

---

## 10. Criterion 8: Esoteric Theory Integration

**Relates to Existing Theories**: YES - Leverages multiple existing theory components.

### Theory Connections:

**1. Cryptographic Signatures (COMP-1.1 Crypto)**
- **Used**: Ed25519 signatures for audit trail verification
- **Integration**: Audit trail logs AppendEntries RPCs with leader signatures
- **Synergy**: Reuses existing `wn_crypto_sign()` and `wn_crypto_verify()` functions
- **Research Opportunity**: None (standard application)

**2. Merkle Proofs (COMP-1.8 Merkle Trees)**
- **Used**: Region membership verification
- **Current Application**: Sheaf overlap messages (topos.c)
- **Proposed Extension**: Could extend to Merkle audit logs (transparency log style)
- **Synergy**: Audit trail could use Merkle tree for tamper-evident log storage
- **Research Opportunity**: **MEDIUM** - "Certificate Transparency for Raft Logs"
  - Each audit log entry gets Merkle inclusion proof
  - Enables external auditors to verify log integrity
  - Aligns with transparency logs discussed in code tampering doc

**3. Capability Security (COMP-3 Capabilities)**
- **Used**: Authorization for consensus operations
- **Current**: Multi-party approval for critical operations (from MULTI_PARTY_CONSENSUS.md)
- **Synergy**: Byzantine leader detection could trigger automatic capability revocation
- **Research Opportunity**: **LOW** - Standard access control application

**4. HLC (Hybrid Logical Clocks) (COMP-1.6 HLC)**
- **Used**: Causality tracking in distributed events
- **Potential Application**: Term limits could use HLC for more precise "leader tenure" measurement
- **Synergy**: Instead of wall-clock 60 seconds, use HLC logical time (more deterministic)
- **Research Opportunity**: **MEDIUM** - "HLC-Based Leader Tenure Limits"
  - Resistance to clock skew attacks
  - More predictable behavior in geo-distributed systems

**5. Operational Semantics (COMP-1.11)**
- **Used**: Replay debugging, race detection
- **Potential Application**: Audit trail could generate operational semantics trace for Byzantine analysis
- **Synergy**: Small-step evaluation of Raft state machine transitions
- **Research Opportunity**: **HIGH** - "Formal Verification of Hybrid Raft+Crypto"
  - Prove that Merkle proofs + signatures prevent Byzantine data corruption
  - Prove audit trail detects Byzantine leader in ≤1 round
  - Use operational semantics framework for proof
  - **Novel contribution**: No existing formal proof of this specific hybrid

**Novel Combinations**:

**Hybrid BFT = Raft + Merkle + Crypto + Audit**
- **Theoretical Gap**: Most literature treats BFT as "all or nothing" (pure PBFT vs. pure Raft)
- **Novel Insight**: This doc proposes a **defense-in-depth** approach:
  - Layer 1: Raft (CFT for coordination)
  - Layer 2: Cryptographic verification (BFT for data integrity)
  - Layer 3: Audit trails (BFT for accountability)
  - Layer 4: Client verification (BFT for client-side safety)
- **Research Value**: **HIGH** - Could publish as "Pragmatic Byzantine Resistance in Enterprise Systems"
  - Empirical comparison: Hybrid vs. Pure PBFT (latency, throughput, complexity)
  - Cost-benefit analysis for different threat models
  - Implementation guidelines for other systems

**Topos Theory + Consensus** (Speculative):
- Sheaf gluing (COMP-1.10) currently used for partition healing
- **Potential**: Could model Byzantine leader as "inconsistent sheaf overlap"
  - Different followers see different "glued" states
  - Sheaf consistency condition violated → Byzantine detection
- **Research Opportunity**: **VERY HIGH** (but speculative, 2+ years)

---

## 11. Key Decisions Required

### Decision 1: Implement v1.0 Mitigations or Defer to v2.0?
**Options**:
- **A**: Implement 3 mitigations before v1.0 launch (20-30 hours, 95% Byzantine resistance)
- **B**: Launch v1.0 with pure Raft, add mitigations in v1.1 (risk: production incidents)
- **C**: Implement full PBFT before v1.0 (100+ hours, 100% Byzantine tolerance, high risk)

**Recommendation**: **Option A**
- **Rationale**: Enterprise deployments expect Byzantine resistance, even if threat model is "trusted internal"
- **Risk Mitigation**: Low implementation cost, validates before production exposure
- **Trade-off**: 3-6 months additional validation time

### Decision 2: Audit Trail Scope - What to Log?
**Options**:
- **A**: Log only AppendEntries RPCs (minimalist, ~64 bytes/entry)
- **B**: Log AppendEntries + RequestVote + Heartbeats (comprehensive, ~200 bytes/entry)
- **C**: Log all Raft messages (exhaustive, ~500 bytes/entry, high storage cost)

**Recommendation**: **Option B**
- **Rationale**: RequestVote logging detects Byzantine election manipulation; Heartbeats too noisy
- **Trade-off**: 3× storage vs. minimalist, but still manageable (< 20 GB/day at 1000 ops/sec)

### Decision 3: Client Verification - Mandatory or Optional?
**Options**:
- **A**: Mandatory for all writes (maximum safety, +10ms latency)
- **B**: Optional per-operation flag (flexibility, client complexity)
- **C**: Enabled only for "critical" operations (requires classification logic)

**Recommendation**: **Option B**
- **Rationale**: Different workloads have different latency/safety trade-offs
- **Implementation**: Add `VERIFY_QUORUM` flag to RPC request options
- **Trade-off**: Client complexity, but preserves performance for non-critical ops

### Decision 4: Leader Term Limit Duration?
**Options**:
- **A**: 30 seconds (aggressive, minimizes Byzantine damage, frequent elections)
- **B**: 60 seconds (balanced, recommended in doc)
- **C**: 120 seconds (lenient, reduces election overhead, longer Byzantine window)
- **D**: Configurable (operational complexity)

**Recommendation**: **Option B** (60 seconds), **make configurable** in v1.1
- **Rationale**: 60 seconds limits Byzantine damage while keeping election overhead < 1%
- **Future**: Make configurable based on operational experience (some deployments may want 30s, others 120s)

### Decision 5: Full PBFT - Now or Later?
**Options**:
- **A**: Implement PBFT in v1.0 (maximum security, 100+ hours, high risk)
- **B**: Defer to v2.0, only if customer demands (pragmatic)
- **C**: Never implement (hybrid approach sufficient)

**Recommendation**: **Option B**
- **Rationale**: No current requirement for full BFT (trusted internal deployments)
- **Trigger**: If customer requires multi-org deployment (untrusted parties)
- **Timeline**: v2.0+ (12-18 months)

---

## 12. Dependencies on Other Files

### Direct Dependencies:

**1. CONSENSUS_CORE_CODE_TAMPERING.MD** (STRONG)
- **Connection**: Code signing prevents malicious code injection that could create Byzantine nodes
- **Integration**: Audit trail depends on signed binaries (otherwise attacker can disable audit)
- **Recommendation**: Implement code signing (Phase 8) BEFORE audit trail (sequential dependency)

**2. MULTI_PARTY_CONSENSUS.md** (MEDIUM)
- **Connection**: Multi-admin approval for critical operations (e.g., rollbacks)
- **Integration**: Byzantine leader detection could auto-trigger approval workflow
- **Example**: "Leader misbehavior detected → require 3-of-5 admin approval to continue"
- **Synergy**: Multi-sig approval + audit trail = comprehensive governance

**3. inter_node_event_auth.md** (WEAK)
- **Connection**: Discusses race conditions and capability security
- **Integration**: Client quorum verification uses same capability auth as inter-node RPC
- **Overlap**: Both discuss 6-gate authentication (signature, expiry, nonce, etc.)
- **Recommendation**: Ensure consistent auth implementation across audit trail and RPC layer

### Implicit Dependencies:

**Wave 4 RPC Layer** (CRITICAL):
- Audit trail needs network layer to query followers
- Client quorum verification IS an RPC feature
- **Timeline**: Audit trail can use local Raft log initially, extend to network queries post-Wave 4

**Phase 1 Crypto** (COMPLETE):
- Ed25519 signatures already implemented
- Audit trail directly uses existing `wn_crypto_sign()` and `wn_crypto_verify()`
- **No blockers**

**Phase 6 Raft** (COMPLETE):
- Raft consensus already implemented
- Audit trail extends Raft log structure
- **No blockers**

---

## 13. Priority Ranking

**Overall Priority**: **P1** (v1.0 enhancement - should do before launch)

### Breakdown by Component:

**Audit Trail**: **P1**
- **Justification**: Enterprise systems require audit logs for compliance (SOC 2, HIPAA, etc.)
- **Effort**: 10-15 hours
- **Risk**: Low
- **Value**: HIGH (detects Byzantine behavior post-facto, enables forensics)

**Client Quorum Verification**: **P1**
- **Justification**: Critical for client-side safety guarantees
- **Effort**: 8-12 hours
- **Risk**: Medium (client-side complexity)
- **Value**: HIGH (prevents client impact from Byzantine leader)

**Leader Term Limits**: **P0** (if time permits) or **P1**
- **Justification**: Simple, high ROI (2-3 hours for significant blast radius reduction)
- **Effort**: 2-3 hours
- **Risk**: Low
- **Value**: MEDIUM (limits damage, doesn't prevent it)

**Full PBFT**: **P2** (v2.0)
- **Justification**: Not needed unless multi-org deployment required
- **Effort**: 100+ hours
- **Risk**: HIGH
- **Value**: HIGH (only if threat model demands it)

### Recommended Implementation Order:

1. **Leader Term Limits** (2-3 hours) - Quick win, immediate value
2. **Audit Trail** (10-15 hours) - Foundation for detection
3. **Client Quorum Verification** (8-12 hours) - Client safety layer
4. **Integration Testing** (40 hours) - All three together
5. **Production Validation** (3-6 months) - Gradual rollout

**Total v1.0 Effort**: **20-30 hours coding + 3-6 months validation** = **Phase 8** (before v1.0 launch)

**Deferral to v2.0**: Full PBFT (only if customer requirement emerges)

---

## Summary Table

| Criterion | Rating | Notes |
|-----------|--------|-------|
| **Architectural Fit** | ✅ YES | Hybrid approach aligns with existing crypto + Raft |
| **NASA Compliance** | ✅ SAFE | All mitigations maintain Power of Ten compliance |
| **v1.0 Timing** | ⚠️ ENHANCEMENT | Recommended before launch, not blocking |
| **Integration Complexity** | 4/10 MEDIUM | 20-30 hours total, well-defined integration points |
| **Theoretical Rigor** | ✅ RIGOROUS | Proven techniques, needs empirical validation |
| **Security Impact** | 🔒 CRITICAL | Reduces Byzantine risk from HIGH to MEDIUM |
| **Resource Cost** | 💰 LOW | < 1% overhead (latency, CPU, storage) |
| **Production Viability** | 🔬 PROTOTYPE | 3-6 months validation needed |
| **Esoteric Theory** | 🎓 HIGH SYNERGY | Leverages Merkle, Crypto, HoTT; research opportunities |
| **Priority** | **P1** | Implement in Phase 8, before v1.0 launch |

---

## Final Recommendation

**Implement the 3-mitigation hybrid approach in Phase 8 (System Security)**:
1. Leader term limits (60 seconds)
2. Audit trail (AppendEntries + RequestVote logging)
3. Client quorum verification (optional per-operation)

**Defer full PBFT to v2.0** unless specific customer requirement emerges.

**Estimated Timeline**:
- Development: 20-30 hours
- Testing: 40-80 hours
- Validation: 3-6 months
- **Total**: Q1-Q2 2025 for v1.0 launch with Byzantine resistance

**This approach provides 95% of PBFT's Byzantine tolerance with 20% of the implementation cost.**
