# ITIL-NEXT Connectors

> External system integrations for ITIL-NEXT.

## Purpose

The Connectors layer handles all external system integrations:
- **MS Graph** - Email read/write (primary communication channel)
- **HubSpot** - Contact/Company enrichment, CRM context
- **CMDB** - Asset links (Valuemation, later phase)

## Integrations

### MS Graph (Priority 1)

Email integration via Microsoft Graph API:
- Read emails from shared mailbox
- Send replies maintaining thread
- Webhook for new message notifications
- Attachment handling

```python
from connectors.msgraph import email

# Read new emails
messages = await email.list_messages(mailbox="support@company.com")

# Send reply
await email.send_reply(ticket, content="We're looking into this...")
```

### HubSpot (Priority 2)

CRM context enrichment:
- Contact lookup by email
- Company tier for SLA assignment
- Deal context (is this a prospect?)

```python
from connectors.hubspot import contacts

# Enrich ticket with CRM context
contact = await contacts.get_by_email(ticket.requester_email)
tier = contact.company.properties.get("tier", "standard")
```

### CMDB (Priority 3 - Later)

Asset management integration:
- CI lookup
- Link ticket to affected assets
- Dependency mapping

## Structure

```
itil-next-connectors/
├── .claude/
│   └── claude.md           # Development context
├── msgraph/
│   ├── __init__.py
│   ├── client.py           # Graph client, auth
│   ├── email.py            # Read/write operations
│   ├── webhooks.py         # Subscription management
│   └── models.py           # Pydantic models
├── hubspot/
│   ├── __init__.py
│   ├── client.py           # HubSpot client
│   ├── contacts.py         # Contact/Company ops
│   └── models.py
├── cmdb/
│   └── (later phase)
├── tests/
│   ├── test_msgraph.py
│   └── test_hubspot.py
└── config.py               # Environment config
```

## Error Handling

```python
# All connectors implement circuit breaker pattern
# Transient errors: retry with backoff
# Permanent errors: fail fast, alert

from connectors.common import with_retry

@with_retry(max_attempts=3)
async def send_email(ticket, content):
    ...
```

## Configuration

```bash
# MS Graph
AZURE_TENANT_ID=...
AZURE_CLIENT_ID=...
AZURE_CLIENT_SECRET=...
SUPPORT_MAILBOX=support@company.com

# HubSpot
HUBSPOT_API_KEY=...

# Redis (for caching)
UPSTASH_REDIS_URL=...
UPSTASH_REDIS_TOKEN=...
```

## Related

- [itil-next](../itil-next) - Orchestrator, vision, prompts
- [itil-next-engine](../itil-next-engine) - Core ticket logic
