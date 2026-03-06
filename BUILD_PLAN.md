# Rails Skills — Full Build Plan

## Structure Per Skill
```
skills/<name>/
├── SKILL.md       # Frontmatter + instructions (<500 lines)
└── reference.md   # Detailed patterns, examples, edge cases
```

## Complete Skill List (28 skills)

### Batch 1: Core Active Record + Routing
1. **migrations** — Herald doc + Rails guide merged
2. **active-record-associations** — Rails guide
3. **active-record-querying** — Rails guide  
4. **routing** — Rails guide

### Batch 2: More Active Record + Controllers
5. **active-record-validations** — Rails guide
6. **active-record-callbacks** — Rails guide
7. **action-controller** — Rails guide (overview + advanced merged)
8. **form-helpers** — Rails guide

### Batch 3: Views + Frontend
9. **stimulus** — Herald doc + Rails guide (working_with_javascript)
10. **turbo** — Rails guide (working_with_javascript)
11. **css-architecture** — Herald docs (css-philosophy + dark-mode)
12. **rails-components** — Herald doc (components)
13. **propshaft** — Herald doc
14. **layouts-and-rendering** — Rails guide

### Batch 4: Other Components
15. **active-job** — Rails guide
16. **action-mailer** — Rails guide
17. **active-storage** — Rails guide
18. **action-cable** — Rails guide
19. **action-text** — Rails guide + Herald lexxy doc
20. **action-mailbox** — Rails guide

### Batch 5: Going Deeper
21. **caching** — Rails guide
22. **security** — Rails guide
23. **testing** — Rails guide (complements minitest skill)
24. **i18n** — Rails guide
25. **active-record-encryption** — Rails guide

### Batch 6: Specialized
26. **uuid-primary-keys** — Herald doc
27. **lucide-icons** — Herald doc
28. **generators** — Rails guide

### Maybe Later / Niche
- **magic-link-auth** — Herald doc (pattern-specific)
- **active-storage-multitenant** — Herald doc (gem-specific)
- **active-record-postgresql** — Rails guide (DB-specific)
- **multiple-databases** — Rails guide
- **composite-primary-keys** — Rails guide
- **api-app** — Rails guide
- **configuring** — Rails guide
- **autoloading** — Rails guide
- **active-support** — Rails guide (massive, might need splitting)
- **active-model** — Rails guide

## Build Notes
- Each SKILL.md: opinionated, agent-focused ("do this, not that"), not a docs dump
- Reference.md: comprehensive patterns, examples, gotchas
- Target Rails 8.1 conventions, deprecate old patterns
- Herald docs provide opinionated starting points where available
- Rails guides provide comprehensive coverage for reference.md
