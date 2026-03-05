# Unit of Work Plan — Auto-Trader (AT)

> **Architecture**: Single-process monolith on Hetzner CPX32 VPS  
> **Deployment**: Git pull + systemd restart (single codebase)  
> **Developer**: Single developer  
> **Decomposition strategy**: 4 sequential development units ordered by dependency graph

---

## Assessment

No user questions required. Rationale:
- Single developer → no team alignment decisions
- Single deployment target → no service boundary decisions  
- Monolith → no inter-service communication decisions
- Q8 deferred project structure to AI → technical decomposition authority delegated
- Dependency matrix in component-dependency.md provides unambiguous build order

## Plan

### Phase 1: Unit Identification
- [x] Analyze component dependency matrix for natural layer boundaries
- [x] Identify leaf components (no dependencies) as foundation layer
- [x] Group components by dependency depth into 4 sequential units
- [x] Verify each unit is independently testable before the next starts

### Phase 2: Mandatory Artifact Generation
- [x] Generate `unit-of-work.md` with unit definitions, responsibilities, and code organization
- [x] Generate `unit-of-work-dependency.md` with build-order dependency matrix
- [x] Generate `unit-of-work-story-map.md` mapping all 34 stories to units
- [x] Document code organization strategy in `unit-of-work.md` (Greenfield)
- [x] Validate unit boundaries and dependencies
- [x] Ensure all stories are assigned to units

### Phase 3: Approval
- [ ] Present completion message
- [ ] Await user approval
