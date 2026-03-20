# DRUID — Rails Implementation

**Repository:** [sld0Ant/rails](https://github.com/sld0Ant/rails) (branch: `main`)

**Conformance:** Level 3 (Full)

## What it is

A DDD fork of Ruby on Rails that replaces MVC with the four-layer DRUID architecture. Zero new gem dependencies — built entirely on ActiveModel, ActiveRecord, and ActionDispatch.

## Artifact mapping

| DRUID artifact | Rails file | Base class |
|---------------|-----------|------------|
| Entity | `app/entities/<name>.rb` | `ApplicationEntity` (ActiveModel::API + Attributes) |
| Record | `app/records/<name>_record.rb` | `ApplicationRecord` (ActiveRecord::Base) |
| Repository | `app/repositories/<name>_repository.rb` | `ApplicationRepository` |
| Service | `app/services/<name>_service.rb` | `ApplicationService` |
| Endpoint | `app/endpoints/<name>_endpoint.rb` | `ApplicationEndpoint` |
| Migration | `db/migrate/XXXXXX_create_<plural>.rb` | ActiveRecord::Migration |

## Quick start

```bash
# Environment
export GEM_HOME="$HOME/.local/share/gem/ruby/3.4.0"
export BUNDLE_PATH="$GEM_HOME"
export PATH="$GEM_HOME/bin:$PATH"

# Create app
cd /path/to/rails-fork
ruby railties/exe/rails new /tmp/myapp --api --dev

# Generate from IR
cd /tmp/myapp
mkdir -p docs
# (place your ir.json in docs/)
bundle exec bin/rails generate from_ir docs/ir.json
bundle exec bin/rails db:migrate

# Routes (config/routes.rb)
#   endpoint CoursesEndpoint
#   endpoint StudentsEndpoint

# Run
bundle exec bin/rails server
```

## App structure

```
app/
├── endpoints/          ApplicationEndpoint (CRUD + HATEOAS + auth + transitions + custom actions)
├── entities/           ApplicationEntity (ActiveModel + transitions DSL + defaults)
├── services/           ApplicationService (perform_transition + guard_transition hook)
├── repositories/       ApplicationRepository (merge-update + paginate + defaults flow)
├── records/            ApplicationRecord (ActiveRecord, hidden behind Repository)
├── actions/            Custom non-CRUD operations (optional)
├── middleware/          SetCurrentUser (X-User-Role/X-User-Id → Current)
└── models/             Current (ActiveSupport::CurrentAttributes)
```

## DRUID features supported

| Feature | Status | Notes |
|---------|--------|-------|
| IR parse + validate | ✅ | `lib/druid/ir.rb`, schema version check |
| Generate from IR | ✅ | `rails generate from_ir docs/ir.json` |
| Extract to IR | ✅ | `rake druid:ir` (round-trip) |
| Attribute defaults | ✅ | IR `default` → entity attribute default |
| Transitions | ✅ | IR `transitions` → entity DSL + auto routes |
| `from:` array | ✅ | Multiple source states |
| Relations | ✅ | belongs_to + has_many, custom FK, optional |
| Self-referential | ✅ | Custom FK + class name patching |
| Collection | ✅ | Sort, filter, search, pagination |
| Unique constraints | ✅ | IR `unique` → migration indexes |
| Custom actions | ✅ | IR `actions` → endpoint config + routes |
| HATEOAS `_links` | ✅ | Self, relations, conditional transitions |
| Auth gate | ✅ | Flat array + hash format, role from header |
| CurrentAttributes | ✅ | `Current.user_role`, `Current.user_id` |
| Error handling | ✅ | 404, 422, 403, 500 JSON responses |
| Instrumentation | ✅ | `ActiveSupport::Notifications` |
| Reserved names | ✅ | Error on Module, Class, etc. |
| OpenAPI emitter | ✅ | `rake druid:emit[openapi]` (implementation-specific extension) |
| Round-trip | ✅ | `rake druid:ir` → `rails g from_ir` → identical |

## Routing DSL

```ruby
Rails.application.routes.draw do
  endpoint CoursesEndpoint                        # full CRUD + transitions + actions
  endpoint CoursesEndpoint, only: [:index, :show] # read-only
  endpoint CoursesEndpoint, except: [:destroy]    # no delete
  endpoint CoursesEndpoint, path: "classes"       # custom path
end
```

One line per resource. Transitions and custom actions auto-routed from entity/endpoint config.
