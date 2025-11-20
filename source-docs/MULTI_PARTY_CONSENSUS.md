● Multi-Party Consensus Mechanisms (m-of-n Approval)

  Absolutely! This is a critical enterprise feature: multi-signature approval workflows (similar to blockchain multi-sig wallets, but for capabilities and operations).

  Let me show you how to implement m-of-n consensus (e.g., "3 of 5 admins must approve").

  ---
  Example 1: Multi-Admin Rollback Approval

  Scenario: Require 3 of 5 admins to approve a production rollback

  Your Setup:
  # Define admin group
  $ worknode create group "Production Admins" --members alice,bob,carol,dave,eve
  Created: group_prod_admins

  # Create rollback operation with 3-of-5 consensus requirement
  $ worknode create approval-policy "Production Rollback" \
      --operation ROLLBACK \
      --require 3-of-5 \
      --approvers group_prod_admins \
      --scope "production/**"
  ✓ Policy created: Any production rollback needs 3 admin signatures

  Data Structure:

  // Multi-party approval policy
  typedef struct {
      uuid_t policy_id;
      char operation_name[128];        // "Production Rollback"
      Capability required_capability;  // CAPABILITY_ROLLBACK
      int required_approvals;          // 3 (m)
      int total_approvers;             // 5 (n)
      uuid_t approver_group;           // group_prod_admins
      char scope_pattern[256];         // "production/**"
      uint64_t timeout_ms;             // How long to wait for approvals
  } ApprovalPolicy;

  // Active approval request
  typedef struct {
      uuid_t request_id;
      ApprovalPolicy* policy;
      uuid_t initiator;                // Who requested the operation
      char operation_details[1024];    // What exactly will be done

      // Approval tracking
      uuid_t approvers[MAX_APPROVERS]; // Who approved (bitset)
      bool approved[MAX_APPROVERS];    // Approval status
      int approval_count;              // Current count

      // Timing
      uint64_t created_at;
      uint64_t deadline;               // created_at + timeout_ms

      // Status
      ApprovalStatus status;           // PENDING, APPROVED, DENIED, EXPIRED
  } ApprovalRequest;

  typedef enum {
      APPROVAL_PENDING,
      APPROVAL_APPROVED,    // m-of-n threshold met
      APPROVAL_DENIED,      // Explicitly denied
      APPROVAL_EXPIRED      // Timeout reached
  } ApprovalStatus;

  ---
  Workflow: Frank Requests Production Rollback

  Frank's Action (engineer, not admin):
  $ worknode rollback production/auth-service --to v2.1.3 --reason "Critical auth bug"
  ⏳ Operation requires 3-of-5 admin approvals
  📨 Approval request sent to: Alice, Bob, Carol, Dave, Eve
  ⏱️  Timeout: 1 hour

  What Happens Internally:

  Result request_rollback(const char* service_path, const char* version,
                          const char* reason, uuid_t requester) {
      // 1. Find approval policy for this operation
      ApprovalPolicy* policy = find_policy_for_operation(
          CAPABILITY_ROLLBACK,
          service_path
      );

      if (!policy) {
          return ERR(ERROR_NO_POLICY, "No approval policy for rollback");
      }

      // 2. Create approval request
      ApprovalRequest* req = approval_request_create();
      req->request_id = uuid_generate();
      req->policy = policy;
      req->initiator = requester; // Frank
      snprintf(req->operation_details, sizeof(req->operation_details),
               "Rollback %s to %s (reason: %s)",
               service_path, version, reason);

      req->created_at = hlc_now();
      req->deadline = hlc_now() + policy->timeout_ms; // 1 hour
      req->approval_count = 0;
      req->status = APPROVAL_PENDING;

      // 3. Get approvers from group
      Worknode* approver_group = get_worknode(policy->approver_group);
      int approver_count = 0;
      for (int i = 0; i < approver_group->assignees.element_count; i++) {
          uuid_t approver_id = approver_group->assignees.elements[i].value;
          req->approvers[approver_count++] = approver_id;
          req->approved[i] = false;
      }
      req->total_approvers = approver_count; // 5

      // 4. Send approval request events to all approvers
      for (int i = 0; i < approver_count; i++) {
          Event* evt = event_create(EVENT_TYPE_APPROVAL_REQUEST);
          evt->target_id = req->approvers[i];
          evt->payload = req;
          event_queue_push(evt);
      }

      // 5. Store pending request
      list_append(&pending_approvals, req);

      // 6. Schedule timeout handler
      timer_schedule(policy->timeout_ms, handle_approval_timeout, req);

      return OK(req);
  }

  Alice, Bob, Carol, Dave, and Eve All Receive:
  ┌─ CRITICAL APPROVAL REQUEST ───────────────────┐
  │ 🚨 Production Rollback Requested              │
  │                                               │
  │ Requester: Frank (Engineer)                   │
  │ Operation: Rollback production/auth-service   │
  │            from v2.3.1 → v2.1.3               │
  │ Reason: "Critical auth bug"                   │
  │                                               │
  │ 🔐 Multi-Signature Required: 3 of 5 admins    │
  │                                               │
  │ Approvals So Far: 0/3                         │
  │ ⏱️  Expires in: 59 minutes                     │
  │                                               │
  │ [Approve] [Deny] [View Details] [Discuss]     │
  └───────────────────────────────────────────────┘

  ---
  Approvals Come In

  Alice Approves (first approval):
  $ worknode approve req_rollback_12345
  ✓ Your approval recorded (1/3 required)

  What Happens:
  Result approve_request(uuid_t request_id, uuid_t approver) {
      // 1. Find request
      ApprovalRequest* req = find_pending_approval(request_id);
      if (!req) {
          return ERR(ERROR_NOT_FOUND, "Approval request not found");
      }

      // 2. Check if approver is in authorized list
      int approver_index = -1;
      for (int i = 0; i < req->total_approvers; i++) {
          if (uuid_equal(req->approvers[i], approver)) {
              approver_index = i;
              break;
          }
      }

      if (approver_index == -1) {
          return ERR(ERROR_UNAUTHORIZED, "Not an authorized approver");
      }

      // 3. Check if already approved
      if (req->approved[approver_index]) {
          return ERR(ERROR_ALREADY_APPROVED, "You already approved this");
      }

      // 4. Record approval
      req->approved[approver_index] = true;
      req->approval_count++;

      // 5. Log approval (audit trail)
      log_audit("Approval: request=%s, approver=%s, count=%d/%d",
                uuid_to_string(request_id),
                uuid_to_string(approver),
                req->approval_count,
                req->policy->required_approvals);

      // 6. Check if threshold met
      if (req->approval_count >= req->policy->required_approvals) {
          // THRESHOLD MET! Execute operation
          req->status = APPROVAL_APPROVED;
          execute_approved_operation(req);

          // Notify all participants
          notify_approval_complete(req);
      } else {
          // Still waiting for more approvals
          notify_approval_progress(req);
      }

      return OK(NULL);
  }

  All Admins See Progress Update:
  ┌─ Approval Progress ───────────────────────────┐
  │ 📊 Rollback Request Status                    │
  │                                               │
  │ Approvals: 1/3 ✅                             │
  │ • Alice ✅ APPROVED                           │
  │ • Bob   ⏳ Pending                            │
  │ • Carol ⏳ Pending                            │
  │ • Dave  ⏳ Pending                            │
  │ • Eve   ⏳ Pending                            │
  │                                               │
  │ ⏱️  Expires in: 52 minutes                     │
  └───────────────────────────────────────────────┘

  Bob Approves (second approval):
  $ worknode approve req_rollback_12345
  ✓ Your approval recorded (2/3 required)

  Carol Approves (third approval - THRESHOLD MET!):
  $ worknode approve req_rollback_12345
  ✅ THRESHOLD MET! 3/5 admins approved
  🚀 Executing rollback now...

  Execution:
  void execute_approved_operation(ApprovalRequest* req) {
      // 1. Verify approval count (double-check)
      assert(req->approval_count >= req->policy->required_approvals);

      // 2. Create multi-signature proof (cryptographic)
      MultiSigProof proof = create_multisig_proof(req);

      // 3. Execute the operation with proof
      Result res = execute_with_proof(req->operation_details, &proof);

      if (is_ok(res)) {
          // 4. Log successful execution
          log_audit("Multi-sig operation executed: request=%s, approvers=%s",
                    uuid_to_string(req->request_id),
                    format_approver_list(req));

          // 5. Notify initiator (Frank)
          Event* evt = event_create(EVENT_TYPE_APPROVAL_COMPLETE);
          evt->target_id = req->initiator;
          evt->payload = res.data;
          event_queue_push(evt);

          // 6. Archive request
          archive_approval_request(req);
      } else {
          // Execution failed even with approvals
          log_error("Multi-sig operation failed: %s", res.error.message);
          notify_approval_execution_failed(req, &res.error);
      }
  }

  Frank Receives Notification:
  ┌─ Rollback Approved & Executed ────────────────┐
  │ ✅ Your rollback request was approved!        │
  │                                               │
  │ Operation: Rollback production/auth-service   │
  │            from v2.3.1 → v2.1.3               │
  │                                               │
  │ Approved by (3/5):                            │
  │ • Alice  ✅ (1 min ago)                       │
  │ • Bob    ✅ (30 sec ago)                      │
  │ • Carol  ✅ (just now)                        │
  │                                               │
  │ 🚀 Rollback executed successfully at 14:32    │
  │ 📊 [View Rollback Status]                     │
  └───────────────────────────────────────────────┘

  ---
  Example 2: Cryptographic Multi-Signature Proof

  Critical: The system generates a cryptographic proof that m-of-n approvers signed off.

  Data Structure:

  // Cryptographic multi-signature proof
  typedef struct {
      uuid_t request_id;
      int required_approvals;        // 3
      int total_approvals;           // 5

      // Approver signatures (Ed25519)
      struct {
          uuid_t approver_id;
          Signature signature;       // 64 bytes
          uint64_t timestamp;
      } approvals[MAX_APPROVERS];

      int approval_count;            // 3 (actual approvals received)

      // Hash of operation details (SHA-256)
      Hash operation_hash;

      // Combined signature (Schnorr multi-sig)
      Signature combined_signature;
  } MultiSigProof;

  Creation:

  MultiSigProof create_multisig_proof(ApprovalRequest* req) {
      MultiSigProof proof = {0};
      proof.request_id = req->request_id;
      proof.required_approvals = req->policy->required_approvals;
      proof.total_approvals = req->total_approvers;

      // 1. Hash the operation details
      proof.operation_hash = wn_crypto_hash(
          req->operation_details,
          strlen(req->operation_details)
      );

      // 2. Collect individual signatures
      int sig_count = 0;
      for (int i = 0; i < req->total_approvers && i < MAX_APPROVERS; i++) {
          if (req->approved[i]) {
              // Get approver's private key (from secure storage)
              Worknode* approver = get_worknode(req->approvers[i]);
              PrivateKey* sk = get_user_private_key(req->approvers[i]);

              // Sign operation hash
              Signature sig;
              wn_crypto_sign(&sig,
                            proof.operation_hash.bytes,
                            sizeof(Hash),
                            *sk);

              proof.approvals[sig_count].approver_id = req->approvers[i];
              proof.approvals[sig_count].signature = sig;
              proof.approvals[sig_count].timestamp = hlc_now();
              sig_count++;
          }
      }

      proof.approval_count = sig_count;

      // 3. Create combined Schnorr signature (optional)
      proof.combined_signature = schnorr_aggregate(
          proof.approvals,
          proof.approval_count
      );

      return proof;
  }

  Verification (can be done by anyone, anytime):

  bool verify_multisig_proof(MultiSigProof* proof) {
      // 1. Check threshold
      if (proof->approval_count < proof->required_approvals) {
          return false; // Not enough approvals
      }

      // 2. Verify each individual signature
      for (int i = 0; i < proof->approval_count; i++) {
          uuid_t approver_id = proof->approvals[i].approver_id;
          Signature* sig = &proof->approvals[i].signature;

          // Get approver's public key
          Worknode* approver = get_worknode(approver_id);
          PublicKey* pk = &approver->public_key;

          // Verify signature over operation hash
          bool valid = wn_crypto_verify(*sig,
                                        proof->operation_hash.bytes,
                                        sizeof(Hash),
                                        *pk);

          if (!valid) {
              return false; // Invalid signature
          }
      }

      // 3. All signatures valid
      return true;
  }

  Audit Trail (immutable log):
  $ worknode audit req_rollback_12345
  ════════════════════════════════════════════════
  MULTI-SIGNATURE APPROVAL AUDIT TRAIL
  ════════════════════════════════════════════════
  Request ID: req_rollback_12345
  Operation:  Rollback production/auth-service v2.3.1 → v2.1.3
  Initiator:  Frank (frank_uuid)
  Policy:     3-of-5 Production Admins
  Created:    2025-01-15 14:00:00 UTC

  Approvals (3/5):
    1. Alice  ✅ 14:01:23 UTC (sig: a3f2...9b1c)
    2. Bob    ✅ 14:02:45 UTC (sig: 7d8e...2a4f)
    3. Carol  ✅ 14:03:12 UTC (sig: e1b9...6c3d)

  Denials (0):
    (none)

  Pending (2):
    • Dave  ⏳ (no response)
    • Eve   ⏳ (no response)

  Execution:
    Status:    APPROVED & EXECUTED
    Executed:  2025-01-15 14:03:15 UTC
    Result:    SUCCESS
    Duration:  3 min 15 sec

  Multi-Sig Proof:
    Operation Hash: 8f3a2b...7e9c (SHA-256)
    Combined Sig:   9a1f3e...4b2d (Schnorr aggregate)
    Verifiable:     ✅ YES (anyone can verify)

  ════════════════════════════════════════════════

  ---
  Example 3: Granting Capabilities with Multi-Party Consensus

  Scenario: Promoting someone to admin requires 4-of-7 current admin consensus

  Your Setup:
  # Create policy for admin promotion
  $ worknode create approval-policy "Promote to Admin" \
      --operation GRANT_CAPABILITY \
      --capability ADMIN \
      --require 4-of-7 \
      --approvers group_admins
  ✓ Policy: Granting ADMIN capability needs 4 admin approvals

  Alice Requests Promotion for Frank:
  $ worknode grant company_root --user frank --capability ADMIN
  ⏳ Operation requires 4-of-7 admin approvals
  📨 Approval request sent to: Alice, Bob, Carol, Dave, Eve, Grace, Hank
  ⏱️  Timeout: 24 hours

  What Happens:

  Result grant_capability_with_approval(Worknode* node, uuid_t user,
                                         Capability cap) {
      // 1. Check if this capability requires multi-party approval
      ApprovalPolicy* policy = find_policy_for_capability_grant(cap);

      if (policy) {
          // 2. Create approval request
          ApprovalRequest* req = approval_request_create();
          req->policy = policy;
          snprintf(req->operation_details, sizeof(req->operation_details),
                   "Grant %s capability to user %s",
                   capability_to_string(cap),
                   uuid_to_string(user));

          // 3. Defer actual grant until approved
          req->deferred_operation = grant_capability_deferred;
          req->deferred_args = create_grant_args(node, user, cap);

          // 4. Request approvals
          return request_approval(req);
      } else {
          // No approval needed - grant immediately
          return grant_capability_immediate(node, user, cap);
      }
  }

  // Executed after 4-of-7 approvals
  void grant_capability_deferred(void* args) {
      GrantArgs* grant_args = (GrantArgs*)args;

      // Now actually grant the capability
      capability_grant(grant_args->node,
                      grant_args->user,
                      grant_args->capability);

      log_audit("Capability granted via multi-sig: user=%s, cap=%x",
                uuid_to_string(grant_args->user),
                grant_args->capability);
  }

  Admins Receive Request:
  ┌─ ADMIN PROMOTION REQUEST ─────────────────────┐
  │ 👤 Promote Frank to Administrator             │
  │                                               │
  │ Requested by: Alice                           │
  │ Candidate:    Frank (Engineer, 2 years)       │
  │ Capability:   ADMIN (full system access)      │
  │                                               │
  │ 🔐 Requires: 4 of 7 admin approvals           │
  │                                               │
  │ Approvals So Far: 0/4                         │
  │ ⏱️  Expires in: 23 hours 59 minutes            │
  │                                               │
  │ [Approve] [Deny] [View Frank's History]       │
  └───────────────────────────────────────────────┘

  Approval Tracking:
  # Real-time status
  $ worknode status req_promote_frank

  Request: Promote Frank to Admin
  Status: PENDING (2/4 approvals)

  Approvals:
    ✅ Alice  (initiator & approver, 2 hours ago)
    ✅ Bob    (1 hour ago)
    ⏳ Carol  (pending)
    ⏳ Dave   (pending)
    ⏳ Eve    (pending)
    ⏳ Grace  (pending)
    ⏳ Hank   (pending)

  Time Remaining: 21 hours 58 minutes

  After 4 Approvals:
  ✅ APPROVAL THRESHOLD MET!
  Frank has been promoted to Administrator

  Approved by (4/7):
    • Alice  ✅
    • Bob    ✅
    • Carol  ✅
    • Dave   ✅

  Frank's new capabilities:
    READ | WRITE | ASSIGN | DELETE | ADMIN | ROLLBACK

  Effective immediately.
  Multi-sig proof: [View Cryptographic Proof]

  ---
  Example 4: Denial Mechanism (Explicit Rejection)

  Scenario: 2 denials cancel the request (configurable threshold)

  Policy:
  $ worknode create approval-policy "Production Rollback" \
      --require 3-of-5 approvals \
      --deny-threshold 2 \
      --approvers group_prod_admins
  ✓ Policy: 3 approvals needed, OR 2 denials to reject

  Bob Denies the Rollback:
  $ worknode deny req_rollback_12345 --reason "Rollback unnecessary, bug already fixed in v2.3.2"
  ⚠️  Your denial recorded (1/2 denials needed to reject)

  What Happens:

  Result deny_request(uuid_t request_id, uuid_t denier, const char* reason) {
      ApprovalRequest* req = find_pending_approval(request_id);

      // 1. Check if denier is authorized
      int denier_index = find_approver_index(req, denier);
      if (denier_index == -1) {
          return ERR(ERROR_UNAUTHORIZED, "Not an authorized approver");
      }

      // 2. Record denial
      req->denied[denier_index] = true;
      req->denial_count++;
      req->denial_reasons[denier_index] = strdup(reason);

      log_audit("Denial: request=%s, denier=%s, reason=%s",
                uuid_to_string(request_id),
                uuid_to_string(denier),
                reason);

      // 3. Check if denial threshold met
      if (req->denial_count >= req->policy->deny_threshold) {
          // REQUEST REJECTED!
          req->status = APPROVAL_DENIED;

          // Notify all parties
          notify_approval_denied(req);

          // Archive request
          archive_approval_request(req);
      } else {
          // Still waiting (denials < threshold)
          notify_denial_progress(req);
      }

      return OK(NULL);
  }

  Carol Also Denies (threshold met):
  $ worknode deny req_rollback_12345 --reason "Agree with Bob, v2.3.2 fixes the issue"
  🚫 DENIAL THRESHOLD MET! Request rejected.

  Frank's Notification:
  ┌─ Rollback Request DENIED ─────────────────────┐
  │ ❌ Your rollback request was rejected          │
  │                                               │
  │ Operation: Rollback production/auth-service   │
  │                                               │
  │ Denials (2/2):                                │
  │ • Bob    ❌ "Rollback unnecessary, bug fixed  │
  │             in v2.3.2"                        │
  │ • Carol  ❌ "Agree with Bob, v2.3.2 fixes it" │
  │                                               │
  │ Approvals received (1):                       │
  │ • Alice  ✅ (before denials)                  │
  │                                               │
  │ Suggestion: Deploy v2.3.2 instead of rolling  │
  │             back to v2.1.3                    │
  └───────────────────────────────────────────────┘

  ---
  Example 5: Weighted Voting (Different Approver Weights)

  Scenario: CTO's approval counts as 2 votes, others count as 1

  Policy:
  $ worknode create approval-policy "Critical Infrastructure Change" \
      --require 5-points \
      --approvers alice:2,bob:1,carol:1,dave:1,eve:1,frank:1 \
      --weights "alice=2 (CTO), others=1"
  ✓ Policy: Need 5 approval points (CTO=2, others=1 each)

  Data Structure:

  typedef struct {
      uuid_t approver_id;
      int weight;              // Voting power (1, 2, 3, etc.)
      char role[64];           // "CTO", "Senior Admin", etc.
  } WeightedApprover;

  typedef struct {
      // ... other fields ...
      WeightedApprover approvers[MAX_APPROVERS];
      int required_points;     // 5 (not 5 approvals, 5 POINTS)
      int current_points;      // Sum of approval weights
  } WeightedApprovalRequest;

  Approval Logic:

  Result approve_weighted(uuid_t request_id, uuid_t approver) {
      WeightedApprovalRequest* req = find_weighted_approval(request_id);

      // 1. Find approver's weight
      int weight = 0;
      int approver_index = -1;
      for (int i = 0; i < req->approver_count; i++) {
          if (uuid_equal(req->approvers[i].approver_id, approver)) {
              weight = req->approvers[i].weight;
              approver_index = i;
              break;
          }
      }

      if (approver_index == -1) {
          return ERR(ERROR_UNAUTHORIZED, "Not an authorized approver");
      }

      // 2. Add weighted approval
      req->approved[approver_index] = true;
      req->current_points += weight;

      log_audit("Weighted approval: approver=%s, weight=%d, total_points=%d/%d",
                uuid_to_string(approver), weight,
                req->current_points, req->required_points);

      // 3. Check if threshold met
      if (req->current_points >= req->required_points) {
          // THRESHOLD MET!
          req->status = APPROVAL_APPROVED;
          execute_approved_operation(req);
      }

      return OK(NULL);
  }

  Example Approval Sequence:

  Initial: 0/5 points

  Alice (CTO) approves → +2 points → 2/5 points ⏳
  Bob approves         → +1 point  → 3/5 points ⏳
  Carol approves       → +1 point  → 4/5 points ⏳
  Dave approves        → +1 point  → 5/5 points ✅ THRESHOLD MET!

  (Only needed 4 people because Alice's vote counted double)

  Alternative Scenario:
  Alice (CTO) is unavailable, others must reach consensus:

  Bob approves   → +1 point  → 1/5 points ⏳
  Carol approves → +1 point  → 2/5 points ⏳
  Dave approves  → +1 point  → 3/5 points ⏳
  Eve approves   → +1 point  → 4/5 points ⏳
  Frank approves → +1 point  → 5/5 points ✅ THRESHOLD MET!

  (Needed all 5 non-CTO approvers)

  ---
  Example 6: Quorum-Based Approval (Dynamic Threshold)

  Scenario: Require majority of AVAILABLE admins (not all 7)

  Policy:
  $ worknode create approval-policy "Emergency Rollback" \
      --require MAJORITY \
      --approvers group_admins \
      --quorum-based \
      --availability-window 30m
  ✓ Policy: Majority of admins who respond within 30 min

  How It Works:

  typedef struct {
      // ... other fields ...
      ApprovalQuorumType quorum_type;  // MAJORITY, SUPERMAJORITY, UNANIMOUS
      uint64_t availability_window_ms; // 30 minutes

      // Dynamic tracking
      int responded_count;             // How many responded (approve OR deny)
      int available_count;             // responded_count (determines quorum)
  } QuorumApprovalRequest;

  Result calculate_dynamic_quorum(QuorumApprovalRequest* req) {
      // 1. Wait for availability window to close
      if (hlc_now() < req->created_at + req->availability_window_ms) {
          return ERR(ERROR_QUORUM_PENDING, "Still waiting for responses");
      }

      // 2. Count available approvers (who responded)
      req->available_count = req->responded_count;

      // 3. Calculate required approvals based on quorum type
      int required = 0;
      switch (req->quorum_type) {
          case QUORUM_MAJORITY:
              required = (req->available_count / 2) + 1; // >50%
              break;
          case QUORUM_SUPERMAJORITY:
              required = (req->available_count * 2) / 3; // ≥66%
              break;
          case QUORUM_UNANIMOUS:
              required = req->available_count;           // 100%
              break;
      }

      req->policy->required_approvals = required;

      log_audit("Quorum calculated: available=%d, required=%d (%s)",
                req->available_count, required,
                quorum_type_to_string(req->quorum_type));

      return OK(NULL);
  }

  Example Timeline:

  00:00 - Request created (7 admins in group)
  00:05 - Alice approves  (responded: 1, approved: 1)
  00:10 - Bob approves    (responded: 2, approved: 2)
  00:15 - Carol denies    (responded: 3, approved: 2)
  00:20 - Dave approves   (responded: 4, approved: 3)
  00:30 - Window closes   (responded: 4, approved: 3)

  Quorum Calculation:
    Total admins: 7
    Available (responded): 4 (Alice, Bob, Carol, Dave)
    Eve, Frank, Hank: No response (offline/unavailable)

    Required (majority of available): 4/2 + 1 = 3 approvals
    Actual approvals: 3 ✅

    THRESHOLD MET! (3 of 4 available = 75% majority)

  Without Quorum (would have needed 4 of 7 = 57%, might timeout waiting for offline admins).

  ---
  Example 7: Revocable Approvals (Change Your Mind)

  Scenario: Approver can retract approval before threshold met

  Bob Approves, Then Changes Mind:

  $ worknode approve req_rollback_12345
  ✓ Approval recorded (2/3)

  # 5 minutes later, Bob reconsiders
  $ worknode revoke-approval req_rollback_12345 --reason "Found newer fix in v2.3.3"
  ✓ Approval revoked (1/3 now)

  Implementation:

  Result revoke_approval(uuid_t request_id, uuid_t approver,
                         const char* reason) {
      ApprovalRequest* req = find_pending_approval(request_id);

      // 1. Can only revoke if request still pending
      if (req->status != APPROVAL_PENDING) {
          return ERR(ERROR_TOO_LATE,
                     "Cannot revoke - request already finalized");
      }

      // 2. Find approver
      int approver_index = find_approver_index(req, approver);
      if (approver_index == -1 || !req->approved[approver_index]) {
          return ERR(ERROR_NOT_APPROVED, "You haven't approved this");
      }

      // 3. Revoke approval
      req->approved[approver_index] = false;
      req->approval_count--;

      log_audit("Approval revoked: request=%s, approver=%s, reason=%s",
                uuid_to_string(request_id),
                uuid_to_string(approver),
                reason);

      // 4. Notify participants
      notify_approval_revoked(req, approver, reason);

      return OK(NULL);
  }

  Notification to Others:
  ┌─ Approval Revoked ────────────────────────────┐
  │ ⚠️  Bob revoked their approval                │
  │                                               │
  │ Request: Production Rollback                  │
  │ Reason: "Found newer fix in v2.3.3"           │
  │                                               │
  │ New Status: 1/3 approvals                     │
  │ (was 2/3, now back to 1/3)                    │
  │                                               │
  │ Still need 2 more approvals.                  │
  └───────────────────────────────────────────────┘

  ---
  Summary: Multi-Party Consensus Mechanisms

  | Mechanism           | Example                                | Use Case                               |
  |---------------------|----------------------------------------|----------------------------------------|
  | m-of-n Threshold    | 3-of-5 admins                          | Standard multi-sig operations          |
  | Cryptographic Proof | Ed25519 signatures + Schnorr aggregate | Auditable, verifiable approvals        |
  | Denial Threshold    | 2 denials → reject                     | Fast rejection without waiting for all |
  | Weighted Voting     | CTO vote = 2x                          | Hierarchical authority                 |
  | Quorum-Based        | Majority of AVAILABLE admins           | Handle offline/unavailable approvers   |
  | Revocable Approvals | Change mind before finalized           | Allow reconsideration                  |
  | Timeouts            | 1 hour / 24 hours / custom             | Prevent indefinite blocking            |
  | Deferred Execution  | Grant capability after approval        | Separation of approval & action        |

  Implementation Features:

  1. ✅ Configurable Thresholds: m-of-n, majority, supermajority, unanimous
  2. ✅ Weighted Approvers: Different voting power per user
  3. ✅ Denial Mechanism: Explicit rejection (not just absence of approval)
  4. ✅ Dynamic Quorum: Calculate based on who's available, not total count
  5. ✅ Cryptographic Proofs: Multi-signature with individual + aggregate sigs
  6. ✅ Revocable Approvals: Change mind before threshold met
  7. ✅ Timeout Handling: Auto-reject or auto-approve after deadline
  8. ✅ Audit Trail: Immutable log of all approvals/denials with timestamps
  9. ✅ Real-Time Progress: Live updates to all participants
  10. ✅ Deferred Operations: Separate approval from execution

   This gives you enterprise-grade multi-party consensus for ANY critical operation: rollbacks, admin promotions, capability grants, deletions, budget approvals, etc.