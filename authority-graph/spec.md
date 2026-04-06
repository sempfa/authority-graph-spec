# Authority Graph Specification

**Version:** 0.1.0
**Status:** Draft
**Series:** Architecting Autonomy
**Companion:** Authority Graph Formalization

This specification defines the formal elements, schemas, evaluation rules, and protocols for encoding an authority graph. It is the implementable reference for the Authority Graph Formalization companion paper.

The specification is technology-agnostic. Implementations may use YAML, JSON, protocol buffers, or any structured format that satisfies the schema constraints.

---

## 1. Authority Unit

An authority unit is the fundamental node of the authority graph: a decision right with defined properties.

### 1.1 Schema

```yaml
AuthorityUnit:
  id: string                      # Unique identifier within the graph
  scope:
    decision_type: string         # What class of decisions (e.g. "refund_approval")
    domain: string                # Over what entities (e.g. "transactions")
    conditions: Expression[]      # Under what circumstances (evaluable predicates)
    limits: Constraint[]          # To what extent (thresholds, caps, boundaries)
  delegation:
    granted_by: AuthorityUnitRef  # Reference to the granting authority unit
    contract: DelegationContract  # Terms of the delegation
    re_delegation: boolean        # Whether the grantee may further delegate
    re_delegation_constraints: Constraint[]  # If permitted, under what further limits
  termination:
    expiry: Timestamp | null      # Time-bounded expiry (null = no time limit)
    revocation_triggers: Expression[]  # Conditions that trigger revocation
    context_dependencies: Expression[] # Conditions under which authority lapses
  provenance:
    chain: AuthorityUnitRef[]     # Ordered list from this unit to the root authority
    verifiable: boolean           # Whether the chain can be cryptographically verified
  metadata:
    created: Timestamp
    last_modified: Timestamp
    version: integer
```

### 1.2 Required Fields

All fields in the schema are required except:
- `termination.expiry` (may be null for non-time-bounded authority)
- `delegation.re_delegation_constraints` (empty if re-delegation is false)
- `provenance.verifiable` (defaults to false if not specified)
- `metadata` (implementation-dependent)

### 1.3 Constraints

- `id` must be unique within the graph
- `scope.conditions` must be mechanically evaluable without human interpretation at runtime
- `delegation.granted_by` must reference an existing authority unit in the graph
- `provenance.chain` must be ordered from this unit to the root authority, with each link referencing a valid authority unit
- An authority unit with an empty `provenance.chain` is a root authority; there must be at least one root authority in any valid graph

### 1.4 Validation Rules

- **No ambient authority:** an authority unit must exist in the graph before any action can be evaluated against it. Components cannot acquire authority by capability or configuration alone.
- **Monotonic attenuation:** if this unit was created by delegation, its scope must be equal to or narrower than the granting unit's scope on every dimension. A delegation that widens scope on any dimension is invalid.
- **Provenance completeness:** every non-root authority unit must have a provenance chain that terminates at a root authority. Broken chains are invalid.

---

## 2. Expressions and Constraints

### 2.1 Expression

An evaluable predicate used in scope conditions, termination triggers, and contract invariants. Expressions form trees to support nested conditions.

```yaml
Expression:
  # A leaf expression (field comparison)
  type: "comparison"
  field: string           # The attribute being evaluated (e.g. "transaction.value")
  operator: Operator      # Comparison operator
  value: any              # A literal value, or a FieldRef for relative comparison

  # OR a compound expression (nesting)
  type: "compound"
  operator: "AND" | "OR" | "NOT"
  children: Expression[]  # Nested expressions
```

An expression is either a leaf (comparing a field to a value) or a compound (combining child expressions with AND, OR, or NOT). This supports arbitrary nesting: an AND of ORs, conditions that reference other fields, and negation.

**Leaf operators:**
- `eq` (equals)
- `neq` (not equals)
- `gt`, `gte`, `lt`, `lte` (numeric comparisons)
- `in` (set membership)
- `not_in` (set exclusion)
- `exists` (field is present and non-null)
- `not_exists` (field is absent or null)
- `confirmed` (the value has been verified by an authoritative source)
- `unconfirmed` (the value has not been verified; triggers monotonic reduction)
- `within` (temporal: the field's timestamp is within a Duration of now)
- `field_ref` (relative: compare this field to another field, e.g. "transaction.value < account.credit_limit")

**FieldRef** (for relative comparisons):

```yaml
FieldRef:
  ref_field: string       # The field to compare against (e.g. "account.credit_limit")
```

When `value` is a `FieldRef` instead of a literal, the operator compares the two fields at evaluation time.

**Example: nested condition**

"Transaction value under $5000 AND (account is verified OR transaction is flagged for manual review)":

```yaml
type: "compound"
operator: "AND"
children:
  - type: "comparison"
    field: "transaction.value"
    operator: "lt"
    value: 5000
  - type: "compound"
    operator: "OR"
    children:
      - type: "comparison"
        field: "account.verified"
        operator: "eq"
        value: true
      - type: "comparison"
        field: "transaction.manual_review"
        operator: "eq"
        value: true
```

### 2.2 Constraint

A limit on the extent of authority.

```yaml
Constraint:
  dimension: string       # What is being limited (e.g. "transaction_value")
  type: "max" | "min" | "range" | "enumeration" | "rate"
  value: any              # The limit value
  period: Duration | null # For rate constraints, the time window
```

---

## 3. Edges

### 3.1 Delegation Edge

Authority flows from grantor to grantee with explicit attenuation.

```yaml
DelegationEdge:
  id: string
  from: AuthorityUnitRef         # Granting authority unit
  to: AuthorityUnitRef           # Receiving authority unit
  contract: DelegationContract
  created: Timestamp
```

```yaml
DelegationContract:
  scope_attenuation: ScopeOverride[]  # How scope narrows from grantor to grantee
  restrictions: Expression[]          # What the grantee may NOT do
  re_delegation_permitted: boolean
  re_delegation_max_depth: integer | null  # Maximum further delegation levels
  termination_on_grantor_revocation: "immediate" | "graceful" | "conditional"
  graceful_period: Duration | null    # If graceful, how long the sunset lasts
  conditional_requirements: Expression[] | null  # If conditional, what must hold
```

**Validation:**
- `scope_attenuation` must only narrow; any override that widens a scope dimension is invalid
- `from` and `to` must reference valid authority units
- `re_delegation_max_depth` of 0 is equivalent to `re_delegation_permitted: false`

### 3.2 Composition Edge

Two authority units interacting at a composition seam.

```yaml
CompositionEdge:
  id: string
  parties: AuthorityUnitRef[]     # The authority units that compose (2 or more)
  contract: CompositionContract
  created: Timestamp
```

Schema for `CompositionContract` defined in Section 5.

### 3.3 Precedence Edge

Directional conflict resolution for a specific interaction class.

```yaml
PrecedenceEdge:
  id: string
  governing: AuthorityUnitRef    # The authority unit whose decision prevails
  yielding: AuthorityUnitRef     # The authority unit that defers
  interaction_class: string      # The class of conflict this precedence applies to
  conditions: Expression[]       # Under what conditions this precedence holds
  bidirectional: boolean         # If true, precedence may reverse for other classes
```

**Validation:**
- Precedence is per interaction class, not global. An edge that claims universal precedence is invalid.
- If `bidirectional` is true, a complementary edge for the reverse class should exist.

### 3.4 Scope Overlap Edge (Diagnostic)

Implicit relationship discovered by graph analysis.

```yaml
ScopeOverlapEdge:
  id: string
  units: AuthorityUnitRef[]      # The authority units whose scopes intersect
  overlap_region: Scope          # The intersection of their scope tuples
  resolution: "composition_contract" | "precedence" | "unresolved"
  contract_ref: CompositionEdgeRef | PrecedenceEdgeRef | null
```

**Note:** Scope overlap edges are not authored. They are produced by the graph analysis function (Section 8). An unresolved overlap is a governance finding requiring a design decision.

---

## 4. Subgraphs: Authority Domains

```yaml
AuthorityDomain:
  id: string
  name: string
  description: string
  authority_units: AuthorityUnitRef[]   # Units belonging to this domain
  domain_contract: DomainContract       # Domain-level governance rules
  parent_constitution: ConstitutionRef  # The constitution this domain inherits from
```

```yaml
DomainContract:
  inherits_from: ConstitutionRef
  invariants: Expression[]              # Domain-specific invariants (additive to constitution)
  overrides: Override[]                 # Constitutional rules this domain modifies
  delegation_policy: DelegationPolicy   # Domain-wide delegation rules
  export_policy: ExportPolicy           # What this domain exposes to composition
```

```yaml
ExportPolicy:
  composable_units: AuthorityUnitRef[]  # Which units are available for cross-domain composition
  composition_primitives_permitted: Primitive[]  # Which primitives other domains may use
  sovereign_retention: string[]         # Decision types this domain never cedes
```

---

## 5. Composition Contract

The governance instrument between composing authority domains.

```yaml
CompositionContract:
  id: string
  parties: AuthorityUnitRef[]           # The composing authority units or domains
  interactions:                         # May define different rules per interaction class
    - interaction_class: string
      primitive: "conjunction" | "disjunction" | "delegation" | "precedence"
      invariants: Expression[]          # Properties that must survive this composition
      conflict_resolution: ConflictResolution
      stop_rights: AuthorityUnitRef[]   # Who can unilaterally deny
  scope: Scope                          # Domain of interactions this contract governs
  sovereign_retention:                  # Per-party non-negotiable authority
    - party: AuthorityUnitRef
      retained_decisions: string[]
  termination:
    expiry: Timestamp | null
    renegotiation_conditions: Expression[]
  metadata:
    created: Timestamp
    version: integer
    approved_by: string[]               # Named authors of the contract
```

```yaml
ConflictResolution:
  strategy: "halt_and_escalate" | "default_deny" | "precedence_resolution"
  escalation_path: EscalationPath | null   # If halt_and_escalate
  precedence_ordering: PrecedenceEdgeRef[] | null  # If precedence_resolution
  timeout: Duration | null                 # Maximum time before default_deny applies
```

```yaml
EscalationPath:
  target: string                    # Who receives the escalation (role, not agent)
  context_required: string[]        # What information must accompany the escalation
  resolution_encoding: boolean      # Whether the resolution is encoded back into the contract
```

---

## 6. Constitutional Hierarchy

### 6.1 Global Constitution

```yaml
Constitution:
  id: string
  invariants: Expression[]          # Universal constraints applying to all domains
  default: "deny"                   # Default decision when no rule applies
  metadata:
    created: Timestamp
    version: integer
    approved_by: string[]
```

### 6.2 Evaluation Precedence

When an action is evaluated, the governance layer resolves in the following order:

1. **Pairwise contract**: if a composition contract exists between the specific authority units involved, evaluate it first. If the result is `deny` or `halt`, stop; the action is blocked. If the result is `permit`, continue, the permit is provisional.
2. **Domain contract**: if the pairwise contract is silent on this interaction class, or after a provisional pairwise permit, evaluate the relevant domain contract(s). If the result is `deny` or `halt`, stop. If `permit` or silent, continue.
3. **Constitutional review** (mandatory): regardless of the result at lower tiers, evaluate all constitutional invariants against the action. If any invariant is violated, the action is denied; this overrides any provisional permit from pairwise or domain tiers. If all invariants pass, the permit is confirmed.
4. **Default deny**: if no rule at any tier covers the interaction and no authority unit's scope includes the action, the action is denied.

Deny at any tier is final. Permit at any tier below the constitution is provisional until constitutional review passes. This ensures constitutional invariants cannot be bypassed by more specific contracts. The constitution is supreme — pairwise and domain contracts resolve specificity within the constitutional bounds, not above them.

### 6.3 Inheritance Rules

- Domain contracts inherit all constitutional invariants unless explicitly overridden
- Overrides must be declared in the domain contract's `overrides` field
- An override that weakens a constitutional invariant (permits what the constitution denies) requires explicit approval recorded in the domain contract's metadata
- Pairwise contracts inherit from both parties' domain contracts; conflicts between domain contracts at the pairwise level are resolved by the composition contract's `conflict_resolution` strategy

---

## 7. Monotonic Reduction Protocol

### 7.1 Trigger Conditions

Monotonic reduction fires when any of the following conditions are met during scope evaluation:

- A condition in the scope tuple evaluates to `unconfirmed`
- A required attribute is absent from the evaluation context
- Two applicable domain contracts produce conflicting results for the same interaction
- The composition contract's scope does not cover the specific interaction class

### 7.2 Reduction Rules

```yaml
MonotonicReduction:
  trigger: Expression              # The condition that triggered reduction
  reduction_action: "contract_to_minimum" | "deny" | "escalate"
  minimum_scope: Scope | null      # If contracting, the minimum viable scope
  escalation_path: EscalationPath | null  # If escalating
  governance_finding: GovernanceFinding   # Always produced
```

### 7.3 Governance Finding

Every monotonic reduction event produces a governance finding.

```yaml
GovernanceFinding:
  id: string
  timestamp: Timestamp
  trigger: Expression              # What caused the reduction
  action_requested: ActionRequest  # What was being attempted
  reduction_applied: string        # What reduction was applied
  authority_units_involved: AuthorityUnitRef[]
  contracts_evaluated: ContractRef[]
  resolution_status: "pending" | "resolved" | "encoded"
  resolution: Resolution | null    # If resolved, what was decided
```

### 7.4 Resolution Encoding

When a governance finding is resolved (by human judgment or automated analysis), the resolution may be encoded back into the governance layer:

1. The resolution is authored as a new transition, contract amendment, or scope adjustment
2. The amendment is staged as a candidate (not applied to the live graph)
3. The candidate is verified: monotonic attenuation check, compositional reachability check, complexity gate
4. If verification passes, the candidate is promoted to the live graph at a defined transition boundary
5. The previous version is retained for rollback
6. The governance finding is updated to `encoded` status

This is the A/B update protocol. The live graph is never modified directly. All changes are staged, verified, and promoted.

---

## 8. Graph Analysis Functions

### 8.1 Scope Overlap Detection

```
detect_overlaps(graph: AuthorityGraph) → ScopeOverlapEdge[]
```

For every pair of authority units in different domains, compute the intersection of their scope tuples. If the intersection is non-empty, produce a `ScopeOverlapEdge`. Flag overlaps that have no corresponding composition contract or precedence edge as `unresolved`.

### 8.2 Gap Detection

```
detect_gaps(graph: AuthorityGraph, decision_space: DecisionSpace) → Gap[]
```

Given a defined decision space (the set of all decision types the system must handle), identify any decision type not covered by at least one authority unit's scope. Each gap is a structural finding: a decision the system must make where no authority unit holds jurisdiction.

```yaml
Gap:
  decision_type: string
  domain: string
  status: "unresolved" | "accepted" | "assigned"
  assigned_to: AuthorityUnitRef | null
```

### 8.3 Delegation Depth Analysis

```
analyse_delegation_depth(graph: AuthorityGraph) → DelegationReport[]
```

For each authority unit, trace the delegation chain to the root. Report the depth, the attenuation at each level, and whether any link in the chain has termination conditions that could invalidate the downstream chain (cascade risk).

```yaml
DelegationReport:
  unit: AuthorityUnitRef
  chain_depth: integer
  chain: AuthorityUnitRef[]
  attenuation_summary: string      # Human-readable summary of scope narrowing
  cascade_risk: boolean            # True if any upstream unit has termination conditions
  cascade_mode: "immediate" | "graceful" | "conditional" | null
```

### 8.4 Compositional Reachability Check

```
check_reachability(graph: AuthorityGraph, constitution: Constitution) → ReachabilityResult
```

Construct the set of all scope states reachable under every combination of triggers across all authority units and composition contracts. If any reachable state grants authority that exceeds what any single transition or contract was permitted to grant, the check fails.

```yaml
ReachabilityResult:
  passed: boolean
  violations: ReachabilityViolation[]
  state_space_size: integer
  depth_bound: integer             # The operational depth used for bounded checking
```

```yaml
ReachabilityViolation:
  reachable_state: Scope           # The state that violates the invariant
  trigger_sequence: Expression[]   # The sequence of triggers that produces it
  violated_invariant: Expression   # Which invariant is violated
  tier: "constitution" | "domain" | "pairwise"
```

---

## 9. Arbitration Interface

The interface between the authority graph and the enforcement layer (control-surface band).

### 9.1 Arbitration Request

```yaml
ArbitrationRequest:
  requesting_agent: string
  action: string
  context: object                  # Structured context for scope evaluation
  target_domain: string | null     # If cross-domain, the target
  timestamp: Timestamp
```

### 9.2 Arbitration Response

```yaml
ArbitrationResponse:
  decision: "permit" | "deny" | "halt" | "escalate"
  authority_unit: AuthorityUnitRef | null     # The unit that sanctioned (if permit)
  contract_evaluated: ContractRef | null      # The contract applied (if cross-domain)
  delegation_chain: AuthorityUnitRef[]        # Provenance for the decision
  invariants_checked: Expression[]            # Which invariants were evaluated
  legibility_record: LegibilityRecord         # Mandatory: produced for every evaluation
  governance_finding: GovernanceFinding | null # If reduction or escalation occurred
```

### 9.3 Legibility Record

Produced as a mandatory byproduct of every arbitration evaluation. The enforcement layer must not permit an action if the legibility record cannot be produced.

```yaml
LegibilityRecord:
  id: string
  timestamp: Timestamp
  action: string
  requesting_agent: string
  authority_unit: AuthorityUnitRef
  scope_evaluated: Scope
  delegation_chain: AuthorityUnitRef[]
  contracts_traversed: ContractRef[]
  decision: "permit" | "deny" | "halt" | "escalate"
  tier_resolved_at: "pairwise" | "domain" | "constitution" | "default_deny"
  monotonic_reduction_applied: boolean
  reduction_trigger: Expression | null
```

---

## 10. Graph Validity Rules

A valid authority graph must satisfy all of the following:

1. **At least one root authority** with no `delegation.granted_by` reference
2. **All provenance chains terminate** at a root authority
3. **All delegation edges satisfy monotonic attenuation**: no delegated scope wider than its source
4. **All composition edges reference a valid composition contract**
5. **All unresolved scope overlaps are flagged** as governance findings
6. **The constitutional hierarchy is consistent**: no domain contract weakens a constitutional invariant without explicit override declaration
7. **The compositional reachability check passes**: no reachable state violates a constitutional or domain invariant
8. **Every authority unit has a termination model**: at minimum, a revocation trigger (authority that cannot be withdrawn is sovereignty, and sovereignty within a governed system is a contradiction)

A graph that fails any validation rule is not deployable. The failing rules must be resolved before the graph connects to the enforcement layer.

---

## Appendix A: Type Definitions

Types referenced throughout the specification.

```yaml
# References
AuthorityUnitRef: string          # The `id` of an AuthorityUnit in the graph
CompositionEdgeRef: string        # The `id` of a CompositionEdge
PrecedenceEdgeRef: string         # The `id` of a PrecedenceEdge
ContractRef: string               # The `id` of a CompositionContract or DelegationContract
ConstitutionRef: string           # The `id` of a Constitution

# Scope (standalone reusable type)
Scope:
  decision_type: string
  domain: string
  conditions: Expression[]
  limits: Constraint[]

# Scope override (used in delegation attenuation)
ScopeOverride:
  dimension: string               # Which scope dimension is being narrowed
  original: any                   # The grantor's value on this dimension
  attenuated: any                 # The grantee's narrowed value
  # Validation: attenuated must be equal to or narrower than original

# Delegation policy (domain-wide delegation rules)
DelegationPolicy:
  max_depth: integer | null       # Maximum delegation chain depth within the domain
  re_delegation_default: boolean  # Default re-delegation permission for units in this domain
  requires_provenance: boolean    # Whether all delegations must carry verifiable provenance
  cross_domain_delegation: "prohibited" | "with_approval" | "permitted"

# Override (domain contract overriding a constitutional invariant)
Override:
  constitutional_invariant: Expression    # The invariant being overridden
  domain_replacement: Expression          # What the domain substitutes
  justification: string                   # Why the override is necessary
  approved_by: string[]                   # Named approvers

# Action request (what was being attempted when a governance finding was produced)
ActionRequest:
  requesting_agent: string
  action: string
  context: object
  target_domain: string | null
  timestamp: Timestamp

# Resolution (outcome of a governance finding)
Resolution:
  resolved_by: string             # Who resolved it (human name or process ID)
  resolution_type: "permit" | "deny" | "scope_adjustment" | "contract_amendment"
  new_transition: Expression | null  # If a new enumeration transition was authored
  effective_from: Timestamp
  audit_record: string            # Reference to the full resolution record

# Decision space (for gap detection)
DecisionSpace:
  decision_types: string[]        # All decision types the system must handle
  domains: string[]               # All entity domains the system operates over

# Primitive (composition primitive type)
Primitive: "conjunction" | "disjunction" | "delegation" | "precedence"

# Duration
Duration:
  value: integer
  unit: "seconds" | "minutes" | "hours" | "days"

# Timestamp
Timestamp: string                 # ISO 8601 format (e.g. "2026-04-01T10:00:00Z")
```

---

## Appendix B: Minimal Complete Example

A three-agent authority graph satisfying all validity rules. This is the "hello world" for the authority graph: the smallest graph that demonstrates every element.

**Scenario:** A document processing system with two domains. A Review Agent can approve documents. An Archive Agent can archive approved documents. A System Authority grants both.

### Constitution

```yaml
Constitution:
  id: "system-constitution"
  invariants:
    - type: "comparison"
      field: "action.irreversible"
      operator: "eq"
      value: true
      # Compound: requires audit trail
    - type: "compound"
      operator: "AND"
      children:
        - type: "comparison"
          field: "action.irreversible"
          operator: "eq"
          value: true
        - type: "comparison"
          field: "audit.record_produced"
          operator: "eq"
          value: true
  default: "deny"
  metadata:
    created: "2026-04-01T00:00:00Z"
    version: 1
    approved_by: ["system-owner"]
```

### Authority Units

```yaml
# Root authority
AuthorityUnit:
  id: "system-authority"
  scope:
    decision_type: "all"
    domain: "documents"
    conditions: []
    limits: []
  delegation:
    granted_by: null              # Root: no grantor
    contract: null
    re_delegation: true
    re_delegation_constraints: []
  termination:
    expiry: null
    revocation_triggers: []
    context_dependencies: []
  provenance:
    chain: []                     # Empty chain = root authority
    verifiable: false
  metadata:
    created: "2026-04-01T00:00:00Z"
    last_modified: "2026-04-01T00:00:00Z"
    version: 1

# Delegated: Review Agent
AuthorityUnit:
  id: "review-agent-approval"
  scope:
    decision_type: "document_approval"
    domain: "documents"
    conditions:
      - type: "comparison"
        field: "document.status"
        operator: "eq"
        value: "pending_review"
    limits:
      - dimension: "document_classification"
        type: "enumeration"
        value: ["internal", "confidential"]    # Cannot approve "top_secret"
        period: null
  delegation:
    granted_by: "system-authority"
    contract:
      scope_attenuation:
        - dimension: "decision_type"
          original: "all"
          attenuated: "document_approval"
        - dimension: "limits"
          original: null
          attenuated: "classification <= confidential"
      restrictions:
        - type: "comparison"
          field: "document.classification"
          operator: "not_in"
          value: ["top_secret"]
      re_delegation_permitted: false
      re_delegation_max_depth: 0
      termination_on_grantor_revocation: "immediate"
      graceful_period: null
      conditional_requirements: null
    re_delegation: false
    re_delegation_constraints: []
  termination:
    expiry: "2026-12-31T23:59:59Z"
    revocation_triggers:
      - type: "comparison"
        field: "agent.status"
        operator: "neq"
        value: "active"
    context_dependencies: []
  provenance:
    chain: ["system-authority"]
    verifiable: false
  metadata:
    created: "2026-04-01T00:00:00Z"
    last_modified: "2026-04-01T00:00:00Z"
    version: 1

# Delegated: Archive Agent
AuthorityUnit:
  id: "archive-agent-archival"
  scope:
    decision_type: "document_archival"
    domain: "documents"
    conditions:
      - type: "comparison"
        field: "document.status"
        operator: "eq"
        value: "approved"
    limits:
      - dimension: "batch_size"
        type: "max"
        value: 100
        period: null
  delegation:
    granted_by: "system-authority"
    contract:
      scope_attenuation:
        - dimension: "decision_type"
          original: "all"
          attenuated: "document_archival"
        - dimension: "conditions"
          original: null
          attenuated: "document must be approved"
      restrictions:
        - type: "comparison"
          field: "document.status"
          operator: "neq"
          value: "approved"
      re_delegation_permitted: false
      re_delegation_max_depth: 0
      termination_on_grantor_revocation: "immediate"
      graceful_period: null
      conditional_requirements: null
    re_delegation: false
    re_delegation_constraints: []
  termination:
    expiry: "2026-12-31T23:59:59Z"
    revocation_triggers:
      - type: "comparison"
        field: "agent.status"
        operator: "neq"
        value: "active"
    context_dependencies: []
  provenance:
    chain: ["system-authority"]
    verifiable: false
  metadata:
    created: "2026-04-01T00:00:00Z"
    last_modified: "2026-04-01T00:00:00Z"
    version: 1
```

### Edges

```yaml
# Delegation edges
DelegationEdge:
  id: "delegation-system-to-review"
  from: "system-authority"
  to: "review-agent-approval"
  contract: # (same as review-agent-approval.delegation.contract above)
  created: "2026-04-01T00:00:00Z"

DelegationEdge:
  id: "delegation-system-to-archive"
  from: "system-authority"
  to: "archive-agent-archival"
  contract: # (same as archive-agent-archival.delegation.contract above)
  created: "2026-04-01T00:00:00Z"

# Composition edge (review and archive interact: archive depends on review's output)
CompositionEdge:
  id: "review-archive-composition"
  parties: ["review-agent-approval", "archive-agent-archival"]
  contract:
    id: "review-archive-contract"
    parties: ["review-agent-approval", "archive-agent-archival"]
    interactions:
      - interaction_class: "archive_after_approval"
        primitive: "precedence"
        invariants:
          - type: "comparison"
            field: "document.approval_record"
            operator: "exists"
            value: null
        conflict_resolution:
          strategy: "default_deny"
          escalation_path: null
          precedence_ordering: ["review-agent-approval"]
          timeout: null
        stop_rights: ["review-agent-approval"]
    scope:
      decision_type: "document_archival"
      domain: "approved_documents"
      conditions:
        - type: "comparison"
          field: "document.status"
          operator: "eq"
          value: "approved"
      limits: []
    sovereign_retention:
      - party: "review-agent-approval"
        retained_decisions: ["approval_status"]
      - party: "archive-agent-archival"
        retained_decisions: ["archival_location"]
    termination:
      expiry: null
      renegotiation_conditions: []
    metadata:
      created: "2026-04-01T00:00:00Z"
      version: 1
      approved_by: ["system-owner"]
  created: "2026-04-01T00:00:00Z"
```

### Validation

This graph satisfies all validity rules:

1. Root authority exists: `system-authority`
2. All provenance chains terminate: both agents chain to `system-authority`
3. Monotonic attenuation: both agents have narrower scope than `system-authority`
4. Composition edge references valid contract: `review-archive-contract`
5. No unresolved scope overlaps: review and archive operate on different decision types
6. Constitutional hierarchy consistent: no overrides declared
7. Reachability check: no reachable state exceeds any single unit's scope
8. All units have termination: expiry + revocation triggers on both agents

---

## Appendix C: Implementation Notes

### Connecting to existing identity

The authority graph does not replace your identity layer. It sits above it.

`requesting_agent` in the ArbitrationRequest is an identity your system already knows: an IAM principal, a service account, an OIDC subject, an agent identifier. The authority graph maps that identity to an authority unit through the delegation chain. The identity layer answers "who is this?" The authority graph answers "what is this identity permitted to decide?"

The binding between identity and authority is the delegation edge. When an IAM role is assigned to an agent, the corresponding authority unit should be created in the graph with a delegation edge from the appropriate supervisor or domain authority. The IAM role grants capability (API access). The delegation edge grants authority (decision rights). Both must exist. Capability without authority is the gap the graph exists to close.

For systems using OAuth or OIDC: the OAuth scope parameter maps to a subset of the authority unit's scope tuple. OAuth scopes are flat strings; authority scopes are structured tuples. The authority graph extends the OAuth model, it does not replace it. The OAuth token proves the delegation occurred. The authority graph specifies what was delegated and under what constraints.

### Naming conventions

Adopt a hierarchical naming convention for `decision_type` values to enable scope comparison and overlap detection:

```
{domain}.{action_class}.{specific_action}
```

Examples:
- `payment.refund.approval`
- `payment.refund.execution`
- `fraud.transaction.hold`
- `fraud.transaction.release`
- `compliance.audit.documentation_hold`

This structure allows overlap detection at any level: two units with `payment.refund.*` decision types overlap at the refund level. A unit with `payment.*` overlaps with all payment units.

For `domain` values, use the organisational boundary, not the technical boundary: `transactions`, `customer_accounts`, `flagged_transactions`, not `payment-service`, `fraud-api`, `compliance-db`.

### Where to start

Recommended order for documenting an existing system:

1. **Define the constitution first.** Name 3-5 universal invariants that apply to every agent. These are the non-negotiable rules: what is always irreversible, what always requires escalation, what no agent can ever do regardless of domain. Start sparse. You can add invariants later through the A/B protocol.

2. **Identify your domains.** Group agents by authority boundary, not deployment boundary. Agents that share a conflict resolution mechanism are in the same domain.

3. **Define root authority.** This is typically the system owner, the platform, or the organisation. The root authority is the source from which all delegation chains originate. In most systems there is one root. In federated systems there may be multiple roots (one per organisation), composed through cross-domain contracts.

4. **Map authority units for each domain.** For each agent, answer the four diagnostic questions: what decisions is it authorised to make, under what conditions, who granted this, when does it expire? If you cannot answer these questions, the agent has implicit authority that must be made explicit.

5. **Draw delegation edges.** For each authority unit, create the delegation edge from its granting authority. Verify monotonic attenuation: the delegated scope must be narrower than or equal to the source.

6. **Run scope overlap detection.** Identify where authority units from different domains intersect. Each unresolved overlap is a governance finding that requires a composition contract.

7. **Write composition contracts for each seam.** Start with conjunction (both must permit) as the default. Use precedence or disjunction only where conjunction produces unacceptable operational constraints.

8. **Validate the graph.** Run all eight validity rules. Fix failures before connecting to the enforcement layer.

### Algorithmic notes on graph analysis

**Scope overlap detection.** Scope tuples contain structured Expressions, not simple numeric ranges. Overlap detection requires evaluating whether two Expression trees can simultaneously be satisfied. For simple conditions (field comparisons against literals), this reduces to range intersection. For compound conditions, use a SAT solver or constraint solver to determine whether the conjunction of both scopes' conditions is satisfiable. If satisfiable, the scopes overlap; the satisfying assignment defines the overlap region.

For most authority graphs (tens to low hundreds of authority units with relatively simple scope conditions), brute-force pairwise comparison with simple constraint evaluation is computationally tractable. Optimise only if the graph exceeds this scale.

**Compositional reachability.** Construct the state space as the product of all authority units' scope dimensions. Each trigger (scope condition becoming true or false) produces a state transition. The reachability check verifies that no reachable state violates a constitutional or domain invariant. For bounded model checking, set the depth bound to the longest plausible sequence of authority transitions within a single operational context (typically 3-10 transitions). Validate the bound against historical incident data: if any known failure mode requires a longer sequence to reproduce, increase the bound.

---

## Versioning

This specification follows semantic versioning. The current version (0.1.0) is a draft aligned with the Authority Graph Formalization companion paper. Breaking changes increment the major version. Additions increment the minor version. Corrections increment the patch version.

Changes to the specification follow the A/B update protocol: candidates are staged, verified against the compositional reachability check, and promoted only after validation passes.

---

## References

- Architecting Autonomy Article 8: The Unit of Authority (authority primitive, six properties)
- Architecting Autonomy Article 9: Authority Composition (composition primitives, contracts)
- Architecting Autonomy Article 10: Legibility as Structural Requirement (legibility properties)
- Architecting Autonomy Article 11: Governance at Machine Speed (control-surface band, independence requirement)
- XACML 3.0 Core Specification (OASIS, 2013): combining algorithms, obligations, PDP/PEP architecture
- NIST SP 800-162: Attribute-Based Access Control (scope evaluation model)
- W3C PROV-DM: Provenance Data Model (delegation, attribution, bundles)
- OAuth 2.0 (RFC 6749): scoped delegation, token lifecycle
- X.509 (RFC 5280): certificate lifecycle, revocation cascade
- Open Policy Agent: policy-as-code evaluation model
