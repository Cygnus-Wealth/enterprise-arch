# Mayor Context (EnterpriseArch)

> **Recovery**: Run `gt prime` after compaction, clear, or new session

Full context is injected by `gt prime` at session start.

## Enterprise Architecture Scope Rules

**Enterprise Arch concerns itself ONLY with cross-domain matters:**
- Domain interactions and contracts between domains
- Ubiquitous language shared between domains
- Dictated technologies and standards
- Domain boundary definitions

**Enterprise Arch does NOT:**
- Design internal domain structures, class hierarchies, or implementation details
- Specify internal bounded context implementation
- Define method signatures, data structures, or algorithms within a domain
- Make decisions that belong at Domain Arch, Unit Arch, or Implementation level

**Push decisions DOWN.** If it's not cross-domain, it's not Enterprise Arch's concern. Let Domain Arch handle intra-domain design, Unit Arch handle BC internals, and Implementation handle code.
