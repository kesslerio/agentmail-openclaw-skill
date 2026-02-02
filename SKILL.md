# AgentMail Skill

**Purpose**: Programmatic email for AI agents via AgentMail API — create inboxes, send/receive messages, manage threads, webhooks, and domains.

**Trigger phrases**: "send email", "create inbox", "check mail", "agentmail", "email agent", "read messages", "email webhook"

## Quick Reference

### Authentication

Requires `AGENTMAIL_API_KEY` environment variable. Get your key from https://agentmail.to

### Core Concepts

- **Inbox**: Email address (e.g., `random123@agentmail.to`) that can send/receive
- **Pod**: Container for multiple inboxes with shared domains
- **Thread**: Email conversation (grouped by subject/references)
- **Message**: Individual email in a thread
- **Draft**: Unsent message that can be edited before sending

### CLI Wrapper

Use the `agentmail-cli` script for common operations:

```bash
# List inboxes
./scripts/agentmail-cli inboxes list

# Create inbox
./scripts/agentmail-cli inboxes create [--username NAME] [--domain DOMAIN]

# Send email
./scripts/agentmail-cli send --inbox-id ID --to "email@example.com" --subject "Hello" --text "Body"

# List messages
./scripts/agentmail-cli messages list --inbox-id ID

# Get message
./scripts/agentmail-cli messages get --inbox-id ID --message-id MSG_ID

# Reply to message
./scripts/agentmail-cli reply --inbox-id ID --message-id MSG_ID --text "Reply body"

# List threads
./scripts/agentmail-cli threads list --inbox-id ID

# Create webhook
./scripts/agentmail-cli webhooks create --url "https://..." --events "message.received"

# List webhooks
./scripts/agentmail-cli webhooks list
```

### Python SDK (Direct Usage)

```python
from agentmail import AgentMail

client = AgentMail(api_key="YOUR_API_KEY")

# Create inbox
inbox = client.inboxes.create()
print(f"Created: {inbox.address}")

# Send message
response = client.inboxes.messages.send(
    inbox_id=inbox.id,
    to=["recipient@example.com"],
    subject="Hello from Agent",
    text="This is the message body",
    html="<p>This is the <b>HTML</b> body</p>"  # optional
)

# List messages in inbox
messages = client.inboxes.messages.list(inbox_id=inbox.id)
for msg in messages:
    print(f"{msg.from_} -> {msg.subject}")

# Reply to a message
client.inboxes.messages.reply(
    inbox_id=inbox.id,
    message_id=message_id,
    text="Thanks for your email!"
)

# Forward a message
client.inboxes.messages.forward(
    inbox_id=inbox.id,
    message_id=message_id,
    to=["another@example.com"]
)
```

### Webhooks for Real-Time Events

```python
# Create webhook for new messages
webhook = client.webhooks.create(
    url="https://your-server.com/webhook",
    event_types=["message.received"]
)

# Webhook payload structure:
# {
#   "event": "message.received",
#   "inbox_id": "...",
#   "message_id": "...",
#   "thread_id": "...",
#   "from": "sender@example.com",
#   "subject": "...",
#   "timestamp": "..."
# }
```

### Pods (Multi-Inbox Management)

```python
# Create pod
pod = client.pods.create(name="my-project")

# Create inbox in pod
inbox = client.pods.inboxes.create(
    pod_id=pod.id,
    username="support",
    domain="agentmail.to"  # or your verified domain
)

# List all inboxes in pod
inboxes = client.pods.inboxes.list(pod_id=pod.id)
```

### Custom Domains

```python
# Register domain
domain = client.domains.create(
    domain="mail.yourdomain.com",
    feedback_enabled=True
)

# Get DNS records to configure
zone_file = client.domains.get_zone_file(domain_id=domain.id)

# Verify domain after DNS setup
client.domains.verify(domain_id=domain.id)
```

### Working with Drafts

```python
# Create draft
draft = client.inboxes.drafts.create(
    inbox_id=inbox_id,
    to=["recipient@example.com"],
    subject="Draft Subject",
    text="Draft body..."
)

# Update draft
client.inboxes.drafts.update(
    inbox_id=inbox_id,
    draft_id=draft.id,
    text="Updated body..."
)

# Send draft
client.inboxes.drafts.send(
    inbox_id=inbox_id,
    draft_id=draft.id
)
```

### Attachments

```python
import base64

# Send with attachment
with open("document.pdf", "rb") as f:
    content = base64.b64encode(f.read()).decode()

client.inboxes.messages.send(
    inbox_id=inbox_id,
    to=["recipient@example.com"],
    subject="Document attached",
    text="Please see attached.",
    attachments=[{
        "filename": "document.pdf",
        "content_type": "application/pdf",
        "content": content
    }]
)

# Get attachment from received message
attachment = client.inboxes.messages.get_attachment(
    inbox_id=inbox_id,
    message_id=message_id,
    attachment_id=attachment_id
)
```

### Labels and Filtering

```python
# List messages with label
messages = client.inboxes.messages.list(
    inbox_id=inbox_id,
    labels=["unread"]
)

# Update message labels
client.inboxes.messages.update(
    inbox_id=inbox_id,
    message_id=message_id,
    add_labels=["processed"],
    remove_labels=["unread"]
)
```

### Metrics

```python
from datetime import datetime, timedelta

# Get inbox metrics
metrics = client.inboxes.metrics.get(
    inbox_id=inbox_id,
    start_timestamp=datetime.now() - timedelta(days=7),
    end_timestamp=datetime.now()
)
```

### Async Client

```python
import asyncio
from agentmail import AsyncAgentMail

async def main():
    client = AsyncAgentMail(api_key="YOUR_API_KEY")
    inbox = await client.inboxes.create()
    await client.inboxes.messages.send(
        inbox_id=inbox.id,
        to=["recipient@example.com"],
        subject="Async Hello",
        text="Sent asynchronously!"
    )

asyncio.run(main())
```

### WebSocket for Real-Time Updates

```python
import threading

with client.websockets.connect() as socket:
    socket.on("message.received", lambda msg: print(f"New: {msg}"))
    
    listener = threading.Thread(target=socket.start_listening, daemon=True)
    listener.start()
    
    # Keep running...
```

## Common Patterns

### Inbox-per-User Pattern
```python
def get_or_create_user_inbox(user_id: str) -> str:
    """Create a dedicated inbox for each user."""
    inbox = client.inboxes.create(
        username=f"user-{user_id}",
        display_name=f"User {user_id}'s Inbox"
    )
    return inbox.id
```

### Poll for New Messages
```python
import time

def poll_inbox(inbox_id: str, callback, interval: int = 60):
    """Poll inbox for new messages."""
    last_check = None
    while True:
        messages = client.inboxes.messages.list(
            inbox_id=inbox_id,
            after=last_check,
            labels=["unread"]
        )
        for msg in messages:
            callback(msg)
        last_check = datetime.now().isoformat()
        time.sleep(interval)
```

### Process and Archive
```python
def process_message(inbox_id: str, message_id: str):
    """Process message and mark as handled."""
    msg = client.inboxes.messages.get(
        inbox_id=inbox_id,
        message_id=message_id
    )
    
    # Do processing...
    
    client.inboxes.messages.update(
        inbox_id=inbox_id,
        message_id=message_id,
        add_labels=["processed"],
        remove_labels=["unread"]
    )
```

## Error Handling

```python
from agentmail.core.api_error import ApiError

try:
    client.inboxes.messages.send(...)
except ApiError as e:
    if e.status_code == 404:
        print("Inbox not found")
    elif e.status_code == 429:
        print("Rate limited, retry later")
    else:
        print(f"Error {e.status_code}: {e.body}")
```

## Installation

```bash
pip install agentmail
```

## Resources

- Docs: https://docs.agentmail.to
- Python SDK: https://github.com/agentmail-to/agentmail-python
- Dashboard: https://agentmail.to
