# Lab 18 — Process Messages with Azure Service Bus

## Lab objective

Create an Azure Service Bus namespace and complete a Python Flask application that demonstrates core messaging patterns for an AI inference workflow. You will send and receive queue messages using **peek-lock** delivery, move malformed messages to the dead-letter queue, and use topic subscriptions with SQL filters to route results by priority.

**Estimated duration:** 30 minutes.

## Prerequisites

- An Azure subscription with permission and quota to provision the required Azure services.
- Visual Studio Code.
- Python 3.12 or later.
- Latest Azure CLI.
- Optional: Ruff extension for VS Code for Python formatting and linting.

**Command convention:** All terminal commands in this guide use Windows PowerShell and are single-line commands. Python function implementations are multiline code to paste into `client/service_bus_functions.py`; they are not shell commands.

---

## Task 1 — Download the starter project and deploy Azure Service Bus

1. Download the official starter ZIP:

   [Download service-bus-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/service-bus-python.zip)

2. Move the ZIP file to your working folder and extract it.
3. Open VS Code. Select **File > Open Folder...** and open the extracted project folder.
4. Open `azdeploy.py`.
5. Change **only** the two values at the top of the script:
   - Resource group name: replace `<your-resource-group-name>` with a dedicated resource group name.
   - Azure region: replace `<your-azure-region>` with a region available to your subscription.
6. Save `azdeploy.py`. Do not modify anything else in the deployment script.
7. In VS Code, select **Terminal > New Terminal**. Use PowerShell in the project root.

Sign in to Azure:

```powershell
az login
```

Follow the browser prompts and select the correct Azure account and subscription.

Register the Azure Service Bus resource provider:

```powershell
az provider register --namespace Microsoft.ServiceBus
```

Start the deployment script:

```powershell
python azdeploy.py
```

When the menu appears, select these options in order:

1. Enter **`1`** — **Create Service Bus namespace**. This creates the resource group if needed and provisions an Azure Service Bus **Standard** namespace.
2. Enter **`2`** — **Create messaging entities**. This creates the `inference-requests` queue, the `inference-results` topic, its subscriptions, and the SQL filter.
3. Enter **`3`** — **Assign role**. This assigns the **Azure Service Bus Data Owner** role to your account so the app can send and receive messages with Microsoft Entra authentication.
4. Enter **`4`** — **Check deployment status**. Confirm the namespace status is `Succeeded`, the messaging entities exist, and the role is assigned. If resources are still provisioning, wait and check again.
5. Enter **`5`** — **Retrieve connection info**. This creates the environment-variable file with the Service Bus fully qualified domain name (FQDN).
6. Enter **`6`** — **Exit**.

The `inference-requests` queue is configured with a **maximum delivery count of 5** and dead-lettering on message expiration. The `inference-results` topic has two subscriptions:

| Subscription | Filter behavior |
|---|---|
| `notifications` | Receives all messages |
| `high-priority` | Receives messages whose application property `priority` equals `high` |

From the project root, load the environment variables:

```powershell
. .\.env.ps1
```

Keep this terminal open. If you close it or open another terminal, load the variables again in the new terminal.

---

## Task 2 — Complete the Service Bus messaging functions

Open `client/service_bus_functions.py` in VS Code. The file contains `BEGIN / END` markers for the function implementations.

**Indentation tip:** Paste each code block flush with the left margin (column 1), select the pasted lines, and press **Tab** to align the block with the markers. Use **Shift+Tab** to outdent if needed. Use spaces consistently. Preserve the starter's existing imports, constants, and `get_client()` helper.

### 2.1 Add code to send messages to the queue

Find `# BEGIN SEND MESSAGES FUNCTION` and add:

```python
def send_messages():
    """Send messages to the queue including one malformed message."""
    client = get_client()
    results = []

    with client:
        with client.get_queue_sender(QUEUE_NAME) as sender:
            # Valid message 1: message_id supports deduplication,
            # correlation_id tracks the inference request, and application_properties hold routing metadata.
            msg1 = ServiceBusMessage(
                body=json.dumps({
                    "prompt": "Extract parties and effective date.",
                    "model": "gpt-4o",
                    "document_id": "doc-001"
                }),
                content_type="application/json",
                message_id=str(uuid.uuid4()),
                correlation_id="req-doc-001",
                application_properties={"priority": "standard", "document_type": "contract"}
            )
            sender.send_messages(msg1)
            results.append({
                "correlation_id": msg1.correlation_id,
                "type": "valid",
                "status": "sent"
            })

            # Valid message 2 is high priority and will match the topic filter in the later exercise.
            msg2 = ServiceBusMessage(
                body=json.dumps({
                    "prompt": "Summarize the key terms.",
                    "model": "gpt-4o",
                    "document_id": "doc-002"
                }),
                content_type="application/json",
                message_id=str(uuid.uuid4()),
                correlation_id="req-doc-002",
                application_properties={"priority": "high", "document_type": "contract"}
            )
            sender.send_messages(msg2)
            results.append({
                "correlation_id": msg2.correlation_id,
                "type": "valid",
                "status": "sent"
            })

            # Intentionally malformed JSON, used to demonstrate dead-lettering.
            msg3 = ServiceBusMessage(
                body="not valid json: [broken",
                content_type="application/json",
                message_id=str(uuid.uuid4()),
                correlation_id="req-doc-003",
                application_properties={"priority": "standard"}
            )
            sender.send_messages(msg3)
            results.append({
                "correlation_id": msg3.correlation_id,
                "type": "malformed",
                "status": "sent"
            })

    return results
```

This function sends three messages individually to `inference-requests`: two valid JSON inference requests and one deliberately malformed payload. Each message includes a unique `message_id`; the valid messages also include correlation IDs and application properties.

Save the file.

### 2.2 Add code to process queue messages with peek-lock

Find `# BEGIN PROCESS MESSAGES FUNCTION` and add:

```python
def process_messages():
    """Receive and process messages from the queue using peek-lock."""
    client = get_client()
    results = []

    with client:
        # Peek-lock is the default mode: a message is locked but remains in the queue until settled.
        with client.get_queue_receiver(queue_name=QUEUE_NAME, max_wait_time=5) as receiver:
            for msg in receiver:
                try:
                    payload = json.loads(str(msg))
                    receiver.complete_message(msg)
                    results.append({
                        "correlation_id": msg.correlation_id,
                        "document_id": payload.get("document_id"),
                        "model": payload.get("model"),
                        "prompt": payload.get("prompt", "")[:50],
                        "status": "completed"
                    })
                except json.JSONDecodeError:
                    receiver.dead_letter_message(
                        msg,
                        reason="MalformedPayload",
                        error_description="Message body is not valid JSON"
                    )
                    results.append({
                        "correlation_id": msg.correlation_id,
                        "document_id": None,
                        "model": None,
                        "prompt": str(msg)[:50],
                        "status": "dead-lettered"
                    })

    return results
```

How it works:

- The receiver uses **peek-lock** delivery. A message is locked while being processed but is not removed until explicitly settled.
- `json.loads()` validates the message body.
- `complete_message()` removes a valid message from the queue.
- `dead_letter_message()` moves invalid JSON to the dead-letter sub-queue and records a reason and error description.
- The queue's maximum delivery count is 5. Unsettled or repeatedly failing messages can be dead-lettered according to the queue's delivery settings.

Save `service_bus_functions.py`.

### 2.3 Add code to inspect the dead-letter queue

Find `# BEGIN INSPECT DLQ FUNCTION` and add:

```python
def inspect_dead_letter_queue():
    """Inspect and remove messages from the dead-letter queue."""
    client = get_client()
    results = []

    with client:
        with client.get_queue_receiver(
            queue_name=QUEUE_NAME,
            sub_queue=ServiceBusSubQueue.DEAD_LETTER,
            max_wait_time=5
        ) as dlq_receiver:
            for msg in dlq_receiver:
                results.append({
                    "message_id": msg.message_id,
                    "correlation_id": msg.correlation_id,
                    "dead_letter_reason": msg.dead_letter_reason,
                    "error_description": msg.dead_letter_error_description,
                    "delivery_count": msg.delivery_count,
                    "body": str(msg)[:100]
                })
                # Completing the dead-letter message removes it from the DLQ.
                dlq_receiver.complete_message(msg)

    return results
```

The function receives from `ServiceBusSubQueue.DEAD_LETTER`, collects diagnostic properties, and completes each inspected message so it is removed from the dead-letter queue.

Save the file.

### 2.4 Add topic messaging with filtered subscriptions

Find `# BEGIN TOPIC MESSAGING FUNCTION` and add:

```python
def topic_messaging():
    """Send messages to a topic and receive from filtered subscriptions."""
    client = get_client()
    sent = []
    notifications = []
    high_priority = []

    with client:
        with client.get_topic_sender(TOPIC_NAME) as sender:
            for i, priority in enumerate(["standard", "high", "standard", "high", "low"]):
                document_id = f"doc-{i + 1:03d}"
                result = {
                    "document_id": document_id,
                    "status": "completed",
                    "confidence": 0.95
                }
                msg = ServiceBusMessage(
                    body=json.dumps(result),
                    content_type="application/json",
                    message_id=str(uuid.uuid4()),
                    application_properties={"priority": priority}
                )
                sender.send_messages(msg)
                sent.append({
                    "document_id": document_id,
                    "priority": priority
                })

        # The notifications subscription has no filter, so it receives all five messages.
        with client.get_subscription_receiver(
            topic_name=TOPIC_NAME,
            subscription_name="notifications",
            max_wait_time=5
        ) as receiver:
            for msg in receiver:
                body = json.loads(str(msg))
                props = msg.application_properties or {}
                priority_val = props.get("priority") or props.get(b"priority", b"unknown")
                if isinstance(priority_val, bytes):
                    priority_val = priority_val.decode("utf-8")
                notifications.append({
                    "document_id": body["document_id"],
                    "priority": priority_val
                })
                receiver.complete_message(msg)

        # The SQL filter on high-priority only matches priority = 'high'.
        with client.get_subscription_receiver(
            topic_name=TOPIC_NAME,
            subscription_name="high-priority",
            max_wait_time=5
        ) as receiver:
            for msg in receiver:
                body = json.loads(str(msg))
                props = msg.application_properties or {}
                priority_val = props.get("priority") or props.get(b"priority", b"unknown")
                if isinstance(priority_val, bytes):
                    priority_val = priority_val.decode("utf-8")
                high_priority.append({
                    "document_id": body["document_id"],
                    "priority": priority_val
                })
                receiver.complete_message(msg)

    return {
        "sent": sent,
        "notifications": notifications,
        "high_priority": high_priority
    }
```

This function sends five messages to the `inference-results` topic with priorities `standard`, `high`, `standard`, `high`, and `low`. It then receives messages from each subscription:

- `notifications` has no filter and receives all five messages.
- `high-priority` receives only the two messages where `priority` equals `high`.

The code handles application property values that may arrive as either strings or bytes. Save the file.

---

## Task 3 — Configure the Python environment

From the project root, switch to the client directory:

```powershell
Set-Location .\client
```

Create a virtual environment:

```powershell
python -m venv .venv
```

Activate it:

```powershell
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks activation, allow it for the current terminal process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the dependencies:

```powershell
pip install -r requirements.txt
```

Verify packages if needed:

```powershell
pip list
```

If the environment variables were loaded in another terminal, load them into the current client terminal:

```powershell
. ..\.env.ps1
```

The prompt should show `(.venv)` when the virtual environment is active.

---

## Task 4 — Run the Flask application and test messaging

From the `client` directory, start the app:

```powershell
python app.py
```

Open a browser and navigate to:

[http://localhost:5000](http://localhost:5000)

The UI provides actions to send queue messages, process them, inspect the dead-letter queue, and test topic subscriptions.

### 4.1 Send queue messages

1. Select **Send Messages**.
2. Confirm that three messages are sent. The results should include correlation IDs and indicate that two messages are valid and one is malformed.
3. The valid messages represent inference requests; the third deliberately contains invalid JSON.

### 4.2 Process messages

1. Select **Process Messages**.
2. Confirm that the receiver processes the queue using peek-lock.
3. Expected outcomes:
   - `req-doc-001` — status `completed`
   - `req-doc-002` — status `completed`
   - `req-doc-003` — status `dead-lettered`
4. The two valid messages should be completed and removed from the main queue. The malformed message should be moved to the dead-letter queue with reason `MalformedPayload`.

### 4.3 Inspect the dead-letter queue

1. Select **Inspect Dead-Letter Queue**.
2. Confirm the malformed message appears with its message ID, correlation ID, dead-letter reason, error description, delivery count, and body preview.
3. The function completes the inspected dead-letter message, removing it from the DLQ.

### 4.4 Send and receive topic messages

1. Select **Send & Receive Topic Messages**.
2. Confirm that five messages are published to the `inference-results` topic.
3. Review the results:
   - The `notifications` subscription should receive **all five** messages.
   - The `high-priority` subscription should receive **two** messages, both with priority `high`.
4. Confirm that the returned document IDs use the `doc-001` through `doc-005` format.

After finishing the tests, stop the Flask app by returning to the PowerShell terminal and pressing **Ctrl+C**.

---

## Task 5 — Clean up Azure resources

Delete the dedicated resource group used for this lab. Replace `<rg-name>` with the actual name configured in `azdeploy.py`:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting the resource group deletes every resource inside it. Use a dedicated resource group; unrelated resources within an existing group will also be deleted.

---

## Troubleshooting

### Verify the Azure Service Bus namespace

- In the Azure portal, open the resource group and confirm the namespace has **Provisioning State: Succeeded**.
- Confirm the namespace tier is **Standard**; topics and subscriptions require Standard tier.
- Run `python azdeploy.py` and choose option **4** to check deployment status.

### Check the queue, topic, subscriptions, and filter

- Ensure `inference-requests` and `inference-results` exist.
- Confirm both `notifications` and `high-priority` subscriptions exist.
- Check that the SQL filter on `high-priority` is `priority = 'high'`.
- If entities are missing, rerun `python azdeploy.py` and select option **2** to create messaging entities.

### Check code placement and indentation

- Ensure all four functions are in `client/service_bus_functions.py` under the corresponding `BEGIN / END` comments.
- Use spaces consistently.
- Preserve the imports, constants, and `get_client()` helper.
- Do not modify code outside the designated sections.

### Verify environment variables

- Confirm `.env` and `.env.ps1` exist in the project root and contain `SERVICE_BUS_FQDN`.
- From the project root, load variables using `. .\.env.ps1`.
- From `client`, load them with `. ..\.env.ps1`.
- Each new terminal session needs its own environment-variable load.

### Check authentication

Verify the active Azure account:

```powershell
az account show
```

- Confirm the **Azure Service Bus Data Owner** role is assigned to your account.
- If role assignment is missing, run `python azdeploy.py` and select option **3** to assign it again.
- Role assignments can take time to propagate; wait and retry if needed.

### Check Python dependencies

- Ensure `(.venv)` appears in the terminal prompt.
- From `client`, run `pip install -r requirements.txt`.
- Use `pip list` to confirm packages installed successfully.

### Queue processing results are unexpected

- Ensure you selected **Send Messages** before **Process Messages**.
- Run processing before inspecting the dead-letter queue.
- Peek-lock messages must be explicitly completed or dead-lettered. Unsettled messages may be redelivered after their lock expires.
- Completing messages in the main queue does not affect the topic subscriptions; they are separate messaging entities.

### Topic filter results are unexpected

- The high-priority subscription should use the SQL filter `priority = 'high'`.
- Make sure the sender sets `priority` as an application property rather than only including it in the JSON body.
- The `notifications` subscription receives all five messages; `high-priority` receives the two high-priority messages.
- Each subscription maintains its own copy and delivery state; completing a message in one subscription does not complete the separate copy in another.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure login completed and `Microsoft.ServiceBus` registered.
- [ ] Standard Service Bus namespace created.
- [ ] `inference-requests` queue and `inference-results` topic created.
- [ ] `notifications` and `high-priority` subscriptions created with the correct filter.
- [ ] Azure Service Bus Data Owner role assigned.
- [ ] Environment variables loaded from `.env.ps1`.
- [ ] `send_messages()` implemented.
- [ ] `process_messages()` implemented using peek-lock, completion, and dead-lettering.
- [ ] `inspect_dead_letter_queue()` implemented.
- [ ] `topic_messaging()` implemented.
- [ ] Python environment configured and dependencies installed.
- [ ] Three queue messages sent and processed.
- [ ] Malformed message found and inspected in the dead-letter queue.
- [ ] Five topic messages sent; all five received by notifications and two by high-priority.
- [ ] Flask stopped and Azure resources deleted.

## Key takeaways

- Azure Service Bus queues decouple producers from consumers and support reliable message processing.
- Peek-lock lets a consumer inspect a message before explicitly completing or dead-lettering it.
- Dead-letter queues preserve diagnostics for messages that cannot be processed.
- Topics and subscriptions fan out messages to multiple consumers.
- SQL filters on application properties route messages based on metadata such as priority.
- The **Azure Service Bus Data Owner** role supports Microsoft Entra-authenticated message operations without embedding a connection string in the app.
