# ITIL-NEXT Connectors Context

## Purpose

The Connectors layer handles all external system integrations:
- **MS Graph**: Email read/write (the primary communication channel)
- **HubSpot**: Contact/Company enrichment, CRM context
- **CMDB**: Asset links (Valuemation integration, later phase)
- **Active Directory**: User/group resolution

## MS Graph Connector (Priority 1)

### Capabilities Needed

```
┌─────────────────────────────────────────────────────────────────┐
│                    MS GRAPH EMAIL OPERATIONS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   READ                                                           │
│   ├── List messages in shared mailbox                           │
│   ├── Get message by ID                                         │
│   ├── Get attachments                                           │
│   ├── Mark as read                                              │
│   └── Subscribe to new messages (webhook)                       │
│                                                                  │
│   WRITE                                                          │
│   ├── Send email (as shared mailbox)                            │
│   ├── Reply to thread                                           │
│   ├── Forward with context                                      │
│   └── Add attachments                                           │
│                                                                  │
│   TRACK                                                          │
│   ├── Link email to ticket (conversationId)                     │
│   ├── Thread tracking (in-reply-to headers)                     │
│   └── Read receipts (if available)                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Authentication

```python
# App-only auth for shared mailbox access
# Requires: Mail.ReadWrite, Mail.Send (Application permissions)

from azure.identity import ClientSecretCredential
from msgraph import GraphServiceClient

credential = ClientSecretCredential(
    tenant_id=config.AZURE_TENANT_ID,
    client_id=config.AZURE_CLIENT_ID,
    client_secret=config.AZURE_CLIENT_SECRET
)

client = GraphServiceClient(credential)
```

### Email → Ticket Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   1. Webhook: New email arrives in support@company.com          │
│                          │                                       │
│   2. Check: Is this a reply to existing ticket?                 │
│             └── conversationId match?                           │
│             └── Subject contains ticket reference?              │
│                          │                                       │
│          ┌───────────────┼───────────────┐                      │
│          ▼               ▼               ▼                      │
│   3a. NEW TICKET    3b. REPLY         3c. FORWARD               │
│       Create            Add to          Parse & create          │
│       ticket            timeline        if from agent           │
│                          │                                       │
│   4. Add TimelineEntry with:                                    │
│      - type: email                                              │
│      - visibility: public                                       │
│      - graph_message_id: for future linking                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Sending Emails

```python
async def send_ticket_email(
    ticket: Ticket,
    content: str,
    from_mailbox: str = "support@company.com"
) -> GraphMessage:
    """
    Send email as ticket owner, from shared mailbox.
    Maintains thread with conversationId.
    """
    message = Message(
        subject=f"RE: [{ticket.reference}] {ticket.subject}",
        body=ItemBody(content=content, content_type="html"),
        to_recipients=[Recipient(email=ticket.requester.email)],
        # Thread linking
        conversation_id=ticket.graph_conversation_id,
        in_reply_to=ticket.latest_customer_email_id
    )
    
    await client.users[from_mailbox].send_mail(message)
```

## HubSpot Connector (Priority 2)

### Capabilities Needed

```
┌─────────────────────────────────────────────────────────────────┐
│                    HUBSPOT OPERATIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CONTACT LOOKUP                                                 │
│   ├── Get contact by email                                      │
│   ├── Get associated company                                    │
│   └── Get contact properties (tier, lifecycle, etc.)            │
│                                                                  │
│   CONTEXT ENRICHMENT                                             │
│   ├── Recent deals (is this a prospect?)                        │
│   ├── Support history (previous tickets)                        │
│   └── Company tier (affects SLA)                                │
│                                                                  │
│   SYNC (optional, phase 3)                                       │
│   ├── Create ticket in HubSpot (mirror)                         │
│   └── Sync timeline activities                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Usage in Engine

```python
# When ticket created, enrich with HubSpot context
async def enrich_ticket(ticket: Ticket) -> TicketContext:
    contact = await hubspot.get_contact_by_email(ticket.requester.email)
    
    if contact:
        company = await hubspot.get_company(contact.company_id)
        return TicketContext(
            customer_tier=company.properties.get("tier", "standard"),
            lifetime_value=company.properties.get("ltv"),
            open_deals=await hubspot.count_open_deals(company.id),
            previous_tickets=await hubspot.count_tickets(contact.id)
        )
    
    return TicketContext.unknown()
```

## Microsoft Teams Connector (Priority 1.5)

### Why Teams > Email

```
EMAIL SPAM PROBLEM:
- Agent gets 50+ ticket notifications/day
- Important buried in noise
- No threading/context
- Delayed reading

TEAMS SOLUTION:
- Real-time notifications
- Threaded conversations
- @mentions for urgency
- Adaptive cards for actions
- Channel-based routing
```

### Capabilities Needed

```
┌─────────────────────────────────────────────────────────────────┐
│                    MS TEAMS OPERATIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   NOTIFICATIONS                                                  │
│   ├── Send to user (1:1 chat)                                   │
│   ├── Send to channel (team updates)                            │
│   ├── Adaptive card (actionable notifications)                  │
│   └── Proactive messaging (bot-initiated)                       │
│                                                                  │
│   ADAPTIVE CARDS                                                 │
│   ├── Ticket summary card                                       │
│   ├── SLA warning card (with snooze action)                     │
│   ├── Escalation request card (with accept/decline)             │
│   └── Reminder card (with complete/snooze)                      │
│                                                                  │
│   CHANNELS                                                       │
│   ├── #support-critical → All critical tickets                  │
│   ├── #support-escalations → Escalation activity                │
│   ├── #support-sla-breaches → SLA breach alerts                 │
│   └── #support-team-{name} → Team-specific                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Options

```python
# Option 1: Incoming Webhook (simple, one-way)
async def send_webhook(channel_webhook_url, card):
    await httpx.post(channel_webhook_url, json=card)

# Option 2: Bot Framework (two-way, richer)
# Requires Azure Bot registration
# Supports proactive messaging, adaptive cards with actions

# Recommendation: Start with webhooks, evolve to bot
```

### Adaptive Card Example

```json
{
  "type": "AdaptiveCard",
  "body": [
    {
      "type": "TextBlock",
      "text": "🔴 SLA Warning: INC-1234",
      "weight": "bolder",
      "size": "medium"
    },
    {
      "type": "FactSet",
      "facts": [
        {"title": "Customer", "value": "Acme Corp (VIP)"},
        {"title": "Priority", "value": "Critical"},
        {"title": "SLA Breach In", "value": "45 minutes"}
      ]
    }
  ],
  "actions": [
    {"type": "Action.OpenUrl", "title": "Open Ticket", "url": "..."},
    {"type": "Action.Submit", "title": "Snooze 30min", "data": {"action": "snooze"}}
  ]
}
```

## CMDB Connector (Priority 3 - Later Phase)

### Capabilities Needed

```
- Asset lookup by name/serial/tag
- Link ticket to configuration item
- Get asset dependencies (for impact analysis)
- Asset owner lookup (for routing)
```

## Directory Structure

```
itil-next-connectors/
├── .claude/
│   └── claude.md           ← This file
├── msgraph/
│   ├── __init__.py
│   ├── client.py           ← Graph client setup
│   ├── email.py            ← Read/write operations
│   ├── webhooks.py         ← Subscription management
│   └── models.py           ← Pydantic models
├── hubspot/
│   ├── __init__.py
│   ├── client.py           ← HubSpot client
│   ├── contacts.py         ← Contact/Company ops
│   └── models.py
├── cmdb/
│   └── (later phase)
├── tests/
│   ├── test_msgraph.py
│   └── test_hubspot.py
└── config.py               ← Environment config
```

## Session State

```yaml
session_id: "connectors-bootstrap"
phase: "design"
progress: 0.1

current_focus: "MS Graph email integration"

decisions_made:
  - App-only auth for shared mailbox
  - Webhook for new email notification
  - ConversationId for thread linking

open_questions:
  - Shared mailbox setup requirements?
  - Rate limiting strategy?
  - Attachment storage (blob or inline)?

next_steps:
  - [ ] MS Graph client scaffold
  - [ ] Email read/list operations
  - [ ] Webhook subscription setup
  - [ ] Send email with thread linking
```

## Environment Config

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # MS Graph
    AZURE_TENANT_ID: str
    AZURE_CLIENT_ID: str
    AZURE_CLIENT_SECRET: str
    SUPPORT_MAILBOX: str = "support@company.com"
    
    # HubSpot
    HUBSPOT_API_KEY: str
    
    # Redis (for caching)
    UPSTASH_REDIS_URL: str
    UPSTASH_REDIS_TOKEN: str
    
    class Config:
        env_file = ".env"
```

## Integration with Engine

```python
# Engine calls connectors, not the other way around

# In engine/services/email_service.py
from connectors.msgraph import email as graph_email

class EmailService:
    async def process_incoming(self, message_id: str):
        """Called by webhook handler."""
        msg = await graph_email.get_message(message_id)
        
        ticket = await self.find_or_create_ticket(msg)
        await self.add_to_timeline(ticket, msg)
        
    async def send_reply(self, ticket_id: str, content: str):
        """Agent sends reply to customer."""
        ticket = await self.ticket_repo.get(ticket_id)
        await graph_email.send_reply(ticket, content)
```
