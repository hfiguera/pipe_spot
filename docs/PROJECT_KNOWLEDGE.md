# PipeSpot - Project Knowledge Base

## Project Overview

**PipeSpot** is an Elixir/Phoenix web application that serves as a contact form system with real-time state management using WebSockets. It's designed as an **embeddable widget** that can be integrated into third-party websites.

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| **Backend** | Elixir with Phoenix Framework 1.7.6 |
| **Database** | PostgreSQL via Ecto |
| **Frontend** | Web Components with Lit + TypeScript |
| **Real-time** | Phoenix Channels + LiveState |
| **HTTP Server** | Bandit |
| **Build Tool** | esbuild |

---

## Key Dependencies

### Backend (mix.exs)

| Dependency | Version | Purpose |
|------------|---------|---------|
| `phoenix` | ~> 1.7.6 | Web framework |
| `phoenix_live_view` | ~> 0.19.0 | Real-time UI |
| `phoenix_ecto` | ~> 4.4 | Ecto integration |
| `ecto_sql` | ~> 3.10 | SQL adapter |
| `postgrex` | >= 0.0.0 | PostgreSQL driver |
| `live_state` | ~> 0.6 | Server-side state management (local: `../live_state`) |
| `cors_plug` | >= 0.0.0 | CORS support |
| `swoosh` | ~> 1.3 | Email delivery |
| `jason` | ~> 1.2 | JSON parsing |
| `finch` | ~> 0.13 | HTTP client |

### Frontend (assets/package.json)

| Dependency | Version | Purpose |
|------------|---------|---------|
| `lit` | ^2.8.0 | Web Components framework |
| `phx-live-state` | ^0.10.1 | LiveState client library |

---

## Project Structure

```
pipe_spot/
├── assets/
│   ├── js/
│   │   ├── app.js              # Phoenix LiveView setup
│   │   ├── contact-form.ts     # Contact form web component
│   │   └── custom_elements.ts  # Web components entry point
│   ├── vendor/
│   │   └── topbar.js           # Loading indicator
│   └── package.json
├── config/
│   ├── config.exs              # Base configuration
│   ├── dev.exs                 # Development config
│   ├── prod.exs                # Production config
│   ├── runtime.exs             # Runtime config
│   └── test.exs                # Test config
├── lib/
│   ├── pipe_spot/
│   │   ├── application.ex      # OTP Application
│   │   ├── contacts/
│   │   │   └── contact.ex      # Contact schema
│   │   ├── contacts.ex         # Contacts context
│   │   ├── mailer.ex           # Email mailer
│   │   └── repo.ex             # Ecto repository
│   ├── pipe_spot_web/
│   │   ├── channels/
│   │   │   ├── contact_form_channel.ex  # Contact form channel
│   │   │   └── live_state_socket.ex     # LiveState socket
│   │   ├── components/
│   │   │   ├── core_components.ex
│   │   │   ├── layouts/
│   │   │   │   ├── app.html.heex
│   │   │   │   └── root.html.heex
│   │   │   └── layouts.ex
│   │   ├── controllers/
│   │   │   ├── error_html.ex
│   │   │   ├── error_json.ex
│   │   │   ├── page_controller.ex
│   │   │   └── page_html/
│   │   │       └── home.html.heex
│   │   ├── endpoint.ex         # HTTP endpoint
│   │   ├── gettext.ex          # i18n
│   │   ├── router.ex           # Routes
│   │   └── telemetry.ex        # Metrics
│   ├── pipe_spot.ex
│   └── pipe_spot_web.ex
├── priv/
│   ├── repo/
│   │   └── migrations/
│   │       └── 20230912133644_create_contacts.exs
│   └── static/
├── test/
├── mix.exs
├── mix.lock
└── pipespot_customer_site.html # Example customer integration
```

---

## Domain Model

### Contact Schema

```elixir
schema "contacts" do
  field :name, :string
  field :email, :string
  field :phone_number, :string

  timestamps()
end
```

**Validations:**
- All fields (`name`, `email`, `phone_number`) are required

### Contacts Context API

```elixir
# List all contacts
Contacts.list_contacts()

# Get a single contact
Contacts.get_contact!(id)

# Create a contact
Contacts.create_contact(%{name: "...", email: "...", phone_number: "..."})

# Update a contact
Contacts.update_contact(contact, %{...})

# Delete a contact
Contacts.delete_contact(contact)

# Get changeset for tracking changes
Contacts.change_contact(contact, attrs \\ %{})
```

---

## Web Layer

### Endpoints & Routes

| Path | Method | Handler | Description |
|------|--------|---------|-------------|
| `/` | GET | `PageController.home/2` | Home page |
| `/live` | WS | Phoenix.LiveView.Socket | LiveView WebSocket |
| `/live_state` | WS | LiveStateSocket | LiveState WebSocket |
| `/dev/dashboard` | GET | LiveDashboard | Metrics dashboard (dev only) |
| `/dev/mailbox` | GET | Swoosh.MailboxPreview | Email preview (dev only) |

### WebSocket Channels

#### LiveStateSocket

- **Path:** `/live_state`
- **Channel:** `contact_form:*` → `ContactFormChannel`

#### ContactFormChannel

Handles real-time contact form state management.

**Initial State:**
```elixir
%{complete: false}
```

**Events:**
| Event | Payload | Action |
|-------|---------|--------|
| `create-contact` | `%{name, email, phone_number}` | Creates contact, sets `complete: true` on success |

---

## Frontend Components

### Contact Form Web Component

**File:** `assets/js/contact-form.ts`

**Tag:** `<contact-form>`

**Attributes:**
| Attribute | Type | Description |
|-----------|------|-------------|
| `url` | string | WebSocket URL (e.g., `ws://localhost:4000/live_state`) |

**State:**
| Property | Type | Description |
|----------|------|-------------|
| `complete` | boolean | Whether form submission succeeded |

**Events Sent:**
- `create-contact` - Dispatched on form submit with form data

**Usage:**
```html
<script type="module" src="http://localhost:4000/assets/custom_elements.js"></script>
<contact-form url="ws://localhost:4000/live_state"></contact-form>
```

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        External Website                              │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │  <contact-form url="ws://localhost:4000/live_state">         │  │
│   │                                                               │  │
│   │   1. User fills form                                          │  │
│   │   2. Submit → dispatch 'create-contact' event                 │  │
│   │                                                               │  │
│   └───────────────────────────┬──────────────────────────────────┘  │
│                               │                                      │
└───────────────────────────────│──────────────────────────────────────┘
                                │ WebSocket
                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│                         PipeSpot Server                                │
│                                                                        │
│   ┌─────────────────────┐    ┌─────────────────────────────────────┐  │
│   │   LiveStateSocket   │───▶│       ContactFormChannel            │  │
│   │   /live_state       │    │                                     │  │
│   └─────────────────────┘    │  3. Receive 'create-contact'        │  │
│                              │  4. Call Contacts.create_contact()  │  │
│                              │  5. Update state: complete = true   │  │
│                              │                                     │  │
│                              └──────────────┬──────────────────────┘  │
│                                             │                          │
│                                             ▼                          │
│                              ┌─────────────────────────────────────┐  │
│                              │         PostgreSQL                  │  │
│                              │         contacts table              │  │
│                              └─────────────────────────────────────┘  │
│                                                                        │
└───────────────────────────────────────────────────────────────────────┘
                                │
                                │ State Update
                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│   6. Client receives state update                                      │
│   7. UI updates to show "Thank you" message                            │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Configuration

### Development Settings

| Setting | Value |
|---------|-------|
| **Host** | localhost |
| **Port** | 4000 |
| **Database** | pipe_spot_dev |
| **DB Username** | postgres |
| **DB Password** | postgres |
| **Code Reloading** | Enabled |
| **CORS** | Enabled (all origins) |

### esbuild Configuration

Two build targets configured:

1. **default** - Standard Phoenix app bundle
   - Entry: `js/app.js`
   - Target: ES2020
   - Output: `priv/static/assets/`

2. **custom_elements** - Web components bundle
   - Entry: `js/custom_elements.ts`
   - Target: ES2020
   - Format: ESM
   - Output: `priv/static/assets/`

---

## Running the Application

### Prerequisites

- Elixir ~> 1.14
- PostgreSQL
- Node.js (for assets)

### Setup

```bash
# Install dependencies
mix setup

# Or manually:
mix deps.get
mix ecto.setup
cd assets && npm install
```

### Development

```bash
# Start the server
mix phx.server

# Or with IEx
iex -S mix phx.server
```

### Testing

```bash
mix test
```

### URLs

- **Main App:** http://localhost:4000
- **LiveDashboard:** http://localhost:4000/dev/dashboard
- **Mailbox:** http://localhost:4000/dev/mailbox

---

## Embeddable Widget Usage

To embed the contact form on an external website:

```html
<!DOCTYPE html>
<html>
<head>
  <script type="module" src="http://localhost:4000/assets/custom_elements.js"></script>
</head>
<body>
  <contact-form url="ws://localhost:4000/live_state"></contact-form>
</body>
</html>
```

**Note:** For production, replace `localhost:4000` with your deployed server URL and use `wss://` for secure WebSocket connections.

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `lib/pipe_spot/contacts.ex` | Business logic for contacts |
| `lib/pipe_spot/contacts/contact.ex` | Contact Ecto schema |
| `lib/pipe_spot_web/channels/contact_form_channel.ex` | Real-time form handling |
| `lib/pipe_spot_web/channels/live_state_socket.ex` | WebSocket configuration |
| `lib/pipe_spot_web/endpoint.ex` | HTTP/WS endpoint |
| `lib/pipe_spot_web/router.ex` | Route definitions |
| `assets/js/contact-form.ts` | Contact form web component |
| `config/config.exs` | Base configuration |
| `config/dev.exs` | Development settings |

---

## Architecture Decisions

1. **LiveState Pattern** - Server-managed state with WebSocket sync enables embedding on external sites while maintaining control over business logic.

2. **Web Components (Lit)** - Framework-agnostic components that work anywhere, perfect for embeddable widgets.

3. **CORS Enabled** - Required for cross-origin embedding on customer websites.

4. **ESM Output** - Modern module format for the custom elements bundle ensures compatibility with `type="module"` script tags.

---

## Future Considerations

- Add authentication for admin dashboard
- Implement contact validation (email format, phone format)
- Add rate limiting for form submissions
- Support multiple form types/topics
- Add email notifications on new contacts
- Implement contact export functionality

