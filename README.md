# Rails Skills

A collection of Ruby on Rails skills for AI coding agents like [OpenCode](https://opencode.ai), Claude Code, and other compatible tools.

## Available Skills

| Skill | Description |
|-------|-------------|
| `action-cable` | Action Cable WebSockets — channels, subscriptions, broadcasting, Solid Cable |
| `action-controller` | Controllers — strong parameters, filters, rendering, redirects, CSRF, sessions |
| `action-mailbox` | Inbound email processing — routing, mailboxes, ingress providers |
| `action-mailer` | Sending emails — mailers, templates, previews, deliver_later, attachments |
| `action-text` | Rich text content — Lexxy/Trix editors, attachments, custom attachables |
| `active-job` | Background jobs — Solid Queue, retries, error handling, serialization |
| `active-model` | Plain Ruby objects with Rails integration — form objects, validations, attributes |
| `active-record-associations` | Model relationships — belongs_to, has_many, polymorphic, through, eager loading |
| `active-record-callbacks` | Lifecycle hooks — when to use (and when NOT to), transaction callbacks |
| `active-record-encryption` | Field-level encryption — deterministic vs non-deterministic, key rotation |
| `active-record-querying` | Database queries — scopes, joins, includes, N+1 prevention, pluck, batching |
| `active-record-validations` | Data validation — built-in validators, custom validators, DB constraint pairing |
| `active-storage` | File uploads — attachments, variants, direct uploads, S3/GCS/Azure |
| `caching` | Caching strategies — fragment, Russian doll, Solid Cache, conditional GET |
| `css-architecture` | Modern CSS — custom properties, @layer, light-dark(), design tokens, components |
| `form-helpers` | Forms — form_with, nested attributes, select helpers, file uploads |
| `generators` | Rails generators — built-in, custom, templates, configuration |
| `i18n` | Internationalization — translations, locale files, pluralization, lazy lookups |
| `layouts-and-rendering` | Views — render vs redirect, layouts, partials with locals, content_for |
| `lucide-icons` | Lucide icon library — lucide-rails gem, SVG icons, accessibility |
| `migrations` | Database migrations — creating tables, columns, indexes, reversible migrations |
| `minitest` | Testing with Minitest — fixtures, assertions, test types, TDD workflow |
| `propshaft` | Asset pipeline — Propshaft (Rails 8 default), CSS organization, import maps |
| `rails-components` | UI components — partials, CSS components, helpers, component patterns |
| `routing` | Routes — resources, nested routes, namespace vs scope, constraints, concerns |
| `security` | Security — CSRF, XSS, SQL injection, credentials, CSP, authentication |
| `stimulus` | Stimulus controllers — targets, values, actions, lifecycle, Turbo integration |
| `testing` | Rails testing ecosystem — test types, system tests, parallel testing, CI |
| `turbo` | Turbo — Drive, Frames, Streams, morphing, broadcasting |
| `uuid-primary-keys` | UUID primary keys — UUIDv7, PostgreSQL/SQLite setup, base36 encoding |

## Installation

Install all skills:

```bash
npx skills add ThinkOodle/rails-skills
```

Install a specific skill:

```bash
npx skills add ThinkOodle/rails-skills --skill migrations
```

## Skill Structure

Each skill contains:
- **`SKILL.md`** — Opinionated, agent-focused instructions (under 500 lines)
- **`reference.md`** — Detailed patterns, examples, and edge cases

Skills are designed to be practical and opinionated — "do this, not that" — rather than exhaustive documentation dumps. They target **Rails 8.1** conventions.

## Contributing

Submit a PR with new skills or improvements to existing ones.

## License

MIT
