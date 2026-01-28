# ITIL-NEXT Connectors: Claude Code Guide

## This Repository

External integrations for ITIL-NEXT. Each connector is independent and follows a common interface.

## Connector Priority

| Priority | Connector | Purpose |
|----------|-----------|---------|
| 1 | MS Graph | Email threading, calendar, users |
| 1.5 | MS Teams | Notifications (replace email spam) |
| 2 | HubSpot | Contact/company sync, CRM |
| 3 | Jira | Issue linking (optional) |

## Architecture

```
connectors/
├── base/                 # Common interfaces
│   ├── connector.py      # BaseConnector abstract class
│   ├── auth.py           # OAuth2, API key handlers
│   └── retry.py          # Retry/backoff logic
│
├── msgraph/              # Microsoft Graph
│   ├── client.py         # Graph API client
│   ├── mail.py           # Email operations
│   ├── calendar.py       # Calendar operations
│   └── users.py          # User/group lookup
│
├── teams/                # Microsoft Teams
│   ├── webhook.py        # Incoming webhooks (simple)
│   ├── bot.py            # Bot Framework (advanced)
│   └── cards.py          # Adaptive card builders
│
├── hubspot/              # HubSpot CRM
│   ├── client.py         # HubSpot API client
│   ├── contacts.py       # Contact sync
│   ├── companies.py      # Company sync
│   └── tickets.py        # Ticket sync (optional)
│
└── webhooks/             # Inbound webhooks
    ├── email.py          # Email webhook receiver
    └── hubspot.py        # HubSpot webhook receiver
```

## Common Interface

```python
from abc import ABC, abstractmethod

class BaseConnector(ABC):
    """All connectors implement this interface."""
    
    @abstractmethod
    async def health_check(self) -> bool:
        """Check if connector is operational."""
        pass
    
    @abstractmethod
    async def authenticate(self) -> None:
        """Establish/refresh authentication."""
        pass
```

## MS Graph Connector

### Email Threading

```python
class GraphMailService:
    async def get_conversation_thread(conversation_id: str) -> List[Message]:
        """Get all messages in an email thread."""
        
    async def reply_to_thread(conversation_id: str, body: str) -> Message:
        """Reply maintaining thread."""
        
    async def create_ticket_from_email(message: Message) -> TicketCreateRequest:
        """Parse email into ticket creation request."""
```

### Key Patterns

```python
# Get email thread for ticket
messages = await graph.mail.get_conversation_thread(
    ticket.msgraph_conversation_id
)

# Reply to customer (maintains thread)
await graph.mail.reply_to_thread(
    conversation_id=ticket.msgraph_conversation_id,
    body=response_html,
    from_address="support@company.com"
)
```

## MS Teams Connector

### Why Teams > Email

```
EMAIL PROBLEMS:
- 50+ notifications/day
- Important buried in spam
- No threading
- Delayed delivery

TEAMS SOLUTION:
- Real-time
- Threaded by channel
- @mentions
- Adaptive cards with actions
```

### Notification Channels

```python
CHANNELS = {
    "critical": "#support-critical",
    "escalations": "#support-escalations", 
    "sla-breaches": "#support-sla-breaches",
    "team-{name}": "#support-team-{name}"
}
```

### Adaptive Cards

```python
class TeamsCardBuilder:
    @staticmethod
    def sla_warning_card(ticket, time_remaining) -> dict:
        """
        Build adaptive card for SLA warning.
        Includes: ticket facts, Open button, Snooze button.
        """
        return {
            "type": "AdaptiveCard",
            "body": [...],
            "actions": [
                {"type": "Action.OpenUrl", "title": "Open Ticket"},
                {"type": "Action.Submit", "title": "Snooze 30min"}
            ]
        }
```

## HubSpot Connector

### Two-Way Sync

```python
class HubSpotSyncService:
    async def sync_contact(contact_id: UUID) -> None:
        """Sync contact to HubSpot, get hubspot_id back."""
        
    async def receive_contact_update(webhook_payload: dict) -> None:
        """Process HubSpot webhook, update local contact."""
```

### Association Pattern

```python
# HubSpot uses associations between objects
# When creating ticket in HubSpot:
{
    "properties": {"subject": "...", "hs_pipeline_stage": "1"},
    "associations": [
        {"to": {"id": contact_hubspot_id}, "types": [...]},
        {"to": {"id": company_hubspot_id}, "types": [...]}
    ]
}
```

## Authentication

### OAuth2 Flow (MS Graph, HubSpot)

```python
class OAuth2Handler:
    async def get_token(self) -> str:
        """Get valid access token, refresh if needed."""
        
    async def refresh_token(self) -> str:
        """Refresh expired token."""
```

### Webhook Security

```python
class WebhookValidator:
    def validate_signature(self, payload: bytes, signature: str) -> bool:
        """Validate webhook signature (HubSpot, Graph)."""
```

## Configuration

```python
# Environment variables (NEVER hardcode)
MSGRAPH_CLIENT_ID=xxx
MSGRAPH_CLIENT_SECRET=xxx  # In secrets manager
MSGRAPH_TENANT_ID=xxx

HUBSPOT_API_KEY=xxx  # In secrets manager

TEAMS_WEBHOOK_URL=xxx  # Per channel
```

## Adding a New Connector

1. Create directory: `connectors/{name}/`
2. Implement `BaseConnector` interface
3. Add auth handler if needed
4. Add to `connectors/__init__.py`
5. Document in this CLAUDE.md

## Testing Connectors

```python
# Use mocked responses
@pytest.fixture
def mock_graph_client():
    with responses.RequestsMock() as rsps:
        rsps.add(
            responses.GET,
            "https://graph.microsoft.com/v1.0/me/messages",
            json={"value": [...]},
            status=200
        )
        yield rsps

# Test with mocks
async def test_get_messages(mock_graph_client):
    messages = await graph.mail.get_messages()
    assert len(messages) > 0
```

## Related Repos

- `itil-next` - Orchestrator, documentation
- `itil-next-engine` - Core business logic (this calls into)
