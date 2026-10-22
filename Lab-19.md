# Lab 19 — Publish and Receive Events with Azure Event Grid

## Lab objective

Deploy an Azure Event Grid Namespace with a namespace topic and filtered event subscriptions, then complete a Python Flask application that publishes content-moderation events and receives them using **pull delivery**.

You will publish CloudEvents, verify event-type filtering, acknowledge successfully processed events, and inspect and reject a test event.

**Estimated duration:** 30 minutes.

## Prerequisites

- An Azure subscription with permission and quota to provision the required resources.
- Visual Studio Code (VS Code).
- Python 3.12 or later.
- The latest Azure CLI.
- Optional: Ruff extension for VS Code for Python formatting and linting.

**Command format:** All terminal commands use Windows PowerShell and are written as one-line commands. Python implementations are multiline code to paste into the specified `.py` file; they are not terminal commands.

---

## Task 1 — Download the starter project and deploy Event Grid

1. Download the official starter ZIP file:

   [Download event-grid-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/event-grid-python.zip)

2. Move the ZIP file to your working folder and extract it.
3. Open VS Code, select **File > Open Folder...**, and open the extracted project folder.
4. Open `azdeploy.py`.
5. Change **only** the two values at the top of the script:
   - Resource group name: replace `<your-resource-group-name>` with a dedicated resource group name.
   - Azure region: replace `<your-azure-region>` with a region available to your subscription.
6. Save the script. Do not change anything else in `azdeploy.py`.
7. In VS Code, select **Terminal > New Terminal**. Use PowerShell in the project root.

Sign in to Azure:

```powershell
az login
```

Follow the browser prompts and select the correct Azure account and subscription.

Register the Event Grid resource provider:

```powershell
az provider register --namespace Microsoft.EventGrid
```

Install the Event Grid Azure CLI extension. The namespace commands used by the deployment script require this extension:

```powershell
az extension add --name eventgrid --yes
```

Start the deployment script:

```powershell
python azdeploy.py
```

When the menu appears, enter **`1`** to select **Create Event Grid namespace and topic**.

This option creates the resource group if required, deploys an Event Grid Namespace using the **Standard** SKU, and creates a namespace topic named `moderation-events` configured for **CloudEvents v1.0** input.

### Create the event subscriptions

At the deployment menu, enter **`2`** to select **Create event subscriptions**.

This creates three pull-delivery subscriptions on the `moderation-events` topic:

| Subscription | Event filter | Purpose |
|---|---|---|
| `sub-flagged` | `com.contoso.ai.ContentFlagged` | Receives flagged content events only |
| `sub-approved` | `com.contoso.ai.ContentApproved` | Receives approved content events only |
| `sub-all-events` | No event-type filter | Receives all topic events; acts as an audit stream |

Each subscription is configured with pull delivery, a **60-second receive lock duration**, a **maximum delivery count of 10**, and a **one-day event time-to-live**.

### Assign roles and retrieve connection details

3. At the deployment menu, enter **`3`** to select **Assign user roles**. The script assigns the **EventGrid Data Sender** and **EventGrid Data Receiver** roles on the namespace so your account can publish and receive events with Microsoft Entra authentication.
4. Enter **`4`** to select **Retrieve connection info**. The script creates `.env` and `.env.ps1` with values including the resource group, namespace name, topic name, and namespace endpoint.
5. Enter **`6`** to exit the deployment script.

If you encounter problems later, run `python azdeploy.py` and select option **`5`** to check deployment status. This verifies the namespace, topic, role assignments, and event subscriptions.

From the project root, load the PowerShell environment variables:

```powershell
. .\.env.ps1
```

Keep this terminal open. If you close it or open another terminal, load the environment variables again in that session.

---

## Task 2 — Complete the event publishing and pull-delivery code

Open `client/event_grid_functions.py` in VS Code. You will add three functions to complete publishing, receiving/acknowledging, and inspecting/rejecting events.

**Indentation tip:** Paste each function flush with the left margin (column 1), select the pasted lines, and press **Tab** to align the block with the existing `BEGIN / END` markers. Use **Shift+Tab** to outdent if necessary. Use spaces consistently.

Preserve the starter's existing imports, constants, and helper functions. In particular, the code relies on the existing `get_publisher_client()`, `get_consumer_client()`, and subscription constants `SUB_FLAGGED`, `SUB_APPROVED`, and `SUB_ALL`.

### 2.1 Publish the five content-moderation events

Find `# BEGIN PUBLISH EVENTS FUNCTION` and add:

```python
def publish_moderation_events():
    """Publish content moderation events to the Event Grid namespace topic."""
    client = get_publisher_client()
    results = []

    # Load event definitions from the JSON file.
    json_path = os.path.join(os.path.dirname(__file__), "moderation_events.json")
    with open(json_path, "r") as f:
        event_definitions = json.load(f)

    # Create CloudEvent objects with a unique id and current UTC timestamp.
    events = []
    for defn in event_definitions:
        defn["data"]["timestamp"] = datetime.now(timezone.utc).isoformat()
        events.append(
            CloudEvent(
                type=defn["type"],
                source=defn["source"],
                subject=defn["subject"],
                data=defn["data"],
                id=str(uuid.uuid4())
            )
        )

    # Publish all events in one request.
    client.send(events)

    for event in events:
        results.append({
            "content_id": event.data["contentId"],
            "event_type": event.type.split(".")[-1],
            "category": event.data["category"],
            "confidence": event.data["confidence"],
            "status": "published"
        })

    return results
```

The `moderation_events.json` file contains five CloudEvent definitions with the envelope fields (`type`, `source`, and `subject`) and their data payload. The function adds a unique event ID and current UTC timestamp to each event, creates CloudEvent objects, and publishes all five events in one call to `send()`.

The publisher client targets the namespace topic endpoint and uses `DefaultAzureCredential` for Microsoft Entra authentication.

Save your changes.

### 2.2 Receive and acknowledge events from all subscriptions

Find `# BEGIN CHECK DELIVERY FUNCTION` and add:

```python
def check_filtered_delivery():
    """Receive and acknowledge events from each subscription to verify filtering."""
    flagged = []
    approved = []
    all_events = []

    # Receive events from the flagged-only subscription.
    consumer = get_consumer_client(SUB_FLAGGED)
    details = consumer.receive(max_events=10, max_wait_time=10)
    tokens = []
    for detail in details:
        event = detail.event
        flagged.append({
            "content_id": event.data.get("contentId"),
            "category": event.data.get("category"),
            "severity": event.data.get("severity"),
            "confidence": event.data.get("confidence")
        })
        tokens.append(detail.broker_properties.lock_token)
    if tokens:
        consumer.acknowledge(lock_tokens=tokens)

    # Receive events from the approved-only subscription.
    consumer = get_consumer_client(SUB_APPROVED)
    details = consumer.receive(max_events=10, max_wait_time=10)
    tokens = []
    for detail in details:
        event = detail.event
        approved.append({
            "content_id": event.data.get("contentId"),
            "category": event.data.get("category"),
            "severity": event.data.get("severity"),
            "confidence": event.data.get("confidence")
        })
        tokens.append(detail.broker_properties.lock_token)
    if tokens:
        consumer.acknowledge(lock_tokens=tokens)

    # Receive all events from the unfiltered audit subscription.
    consumer = get_consumer_client(SUB_ALL)
    details = consumer.receive(max_events=10, max_wait_time=10)
    tokens = []
    for detail in details:
        event = detail.event
        all_events.append({
            "content_id": event.data.get("contentId"),
            "event_type": event.type.split(".")[-1],
            "category": event.data.get("category"),
            "confidence": event.data.get("confidence")
        })
        tokens.append(detail.broker_properties.lock_token)
    if tokens:
        consumer.acknowledge(lock_tokens=tokens)

    return {
        "flagged": flagged,
        "approved": approved,
        "all_events": all_events
    }
```

The function uses an `EventGridConsumerClient` for each subscription and calls `receive()` with a maximum of 10 events and a maximum wait of 10 seconds. Each returned `ReceiveDetails` contains the CloudEvent and a broker lock token.

After processing an event, `acknowledge()` with its lock token removes it from the subscription so it is not delivered again. If an event is not acknowledged, it can be redelivered when its lock expires.

Save your changes.

### 2.3 Inspect and reject a test event

Find `# BEGIN INSPECT AND REJECT FUNCTION` and add:

```python
def inspect_and_reject():
    """Publish one event, inspect its CloudEvent envelope, then reject it."""
    publisher = get_publisher_client()

    # Publish a fresh event so one is available to inspect.
    test_event = CloudEvent(
        type="com.contoso.ai.ContentFlagged",
        source="/services/content-moderation",
        subject="/content/text/test-inspect",
        data={
            "contentId": "test-inspect",
            "contentType": "text",
            "modelName": "text-moderator-v2",
            "modelVersion": "2.4.0",
            "confidence": 0.76,
            "category": "misinformation",
            "severity": "medium",
            "reviewRequired": True,
            "timestamp": datetime.now(timezone.utc).isoformat()
        },
        id=str(uuid.uuid4())
    )
    publisher.send([test_event])

    # Receive the test event from the flagged subscription.
    consumer = get_consumer_client(SUB_FLAGGED)
    details = consumer.receive(max_events=1, max_wait_time=10)

    if not details:
        return None

    detail = details[0]
    event = detail.event
    lock_token = detail.broker_properties.lock_token
    delivery_count = detail.broker_properties.delivery_count

    # Capture the complete CloudEvent envelope and broker delivery count.
    result = {
        "specversion": "1.0",
        "type": event.type,
        "source": event.source,
        "subject": event.subject,
        "id": event.id,
        "time": str(event.time) if event.time else "",
        "data": event.data,
        "delivery_count": delivery_count,
        "action": "rejected"
    }

    # Reject the event because it cannot be processed.
    consumer.reject(lock_tokens=[lock_token])

    return result
```

This function publishes a fresh `ContentFlagged` event to ensure an event is available even if earlier events were acknowledged. It receives that event from `sub-flagged`, captures the full CloudEvent envelope and broker `delivery_count`, then calls `reject()` using the lock token.

**Acknowledge vs. reject:**

- `acknowledge()` confirms successful processing and removes the event from the subscription.
- `reject()` tells Event Grid that processing failed. The event can be discarded or moved to a configured dead-letter destination, depending on the subscription configuration and delivery state.

Save `event_grid_functions.py`. Check that all three functions are between the right markers and have correct indentation. Do not change `client/app.py`; the Flask application already calls these functions and displays their results.

---

## Task 3 — Configure the Python environment

From the project root, change to the `client` directory:

```powershell
Set-Location .\client
```

Create a virtual environment:

```powershell
python -m venv .venv
```

Activate the environment:

```powershell
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks the activation script, allow it for the current process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the dependencies:

```powershell
pip install -r requirements.txt
```

Verify the installed packages if necessary:

```powershell
pip list
```

The terminal prompt should include `(.venv)` when the environment is active. Make sure environment variables are loaded in the same terminal session. From `client`, use:

```powershell
. ..\.env.ps1
```

If environment variables were loaded in another terminal, they will not automatically be available here.

---

## Task 4 — Run the Flask application

From the `client` directory, start the app:

```powershell
python app.py
```

Open a browser and navigate to:

[http://localhost:5000](http://localhost:5000)

The web interface allows you to publish moderation events, receive and acknowledge filtered events, and inspect/reject a test event.

### 4.1 Publish moderation events

1. Select **Publish Moderation Events** in the left panel.
2. The app publishes five events to the `moderation-events` namespace topic: two flagged content events, two approved content events, and one escalated-review event.
3. In the results panel, verify each published event shows its content ID, event type, and category.

The publisher batches the five CloudEvents into one `send()` request.

### 4.2 Receive and acknowledge events

1. Select **Receive & Acknowledge Events**.
2. Review the results for all three subscriptions.

Expected results:

| Subscription | Expected messages | Why |
|---|---:|---|
| **Flagged Subscription** (`sub-flagged`) | 2 | Only `com.contoso.ai.ContentFlagged` events |
| **Approved Subscription** (`sub-approved`) | 2 | Only `com.contoso.ai.ContentApproved` events |
| **All Events Subscription** (`sub-all-events`) | 5 | No event-type filter; receives every event |

The flagged events should have categories indicating policy violations, including **violence** and **hate-speech**. The approved events should have the category **safe**.

The escalated-review event appears only in the all-events subscription because its event type is neither `ContentFlagged` nor `ContentApproved`.

The function acknowledges the received events. If you select **Receive & Acknowledge Events** again without publishing more events, the subscriptions should return zero previously acknowledged events.

### 4.3 Inspect and reject an event

1. Select **Inspect & Reject Event**.
2. The app publishes a new `ContentFlagged` test event, receives it through `sub-flagged`, and displays the complete CloudEvent envelope.
3. Review the `specversion`, `type`, `source`, `subject`, `id`, `time`, data payload, and `delivery_count`.
4. Verify the result shows the action as **rejected**.

Rejecting is different from acknowledging: the event is marked as unprocessable and can be discarded or dead-lettered according to the configured behavior.

When you finish the app exercises, stop Flask by returning to the PowerShell terminal and pressing **Ctrl+C**.

---

## Task 5 — Clean up Azure resources

Delete the dedicated resource group created for this lab. Replace `<rg-name>` with the resource group name you configured in `azdeploy.py`:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting the resource group deletes all resources inside it. Use a dedicated resource group; unrelated resources in an existing group will also be deleted.

---

## Troubleshooting

### Verify the Event Grid Namespace deployment

- In the Azure portal, open the resource group and confirm that the namespace shows **Provisioning State: Succeeded**.
- Verify that the namespace topic `moderation-events` exists.
- Run `python azdeploy.py` and select option **`5`** to check deployment status.

### Check event subscriptions

- Confirm that `sub-flagged`, `sub-approved`, and `sub-all-events` were created and show **Succeeded**.
- If events are not received after publishing, ensure subscriptions were created after the topic. Run `python azdeploy.py` and select option **`2`** to create the subscriptions again if needed.
- Confirm `sub-flagged` filters `com.contoso.ai.ContentFlagged`, `sub-approved` filters `com.contoso.ai.ContentApproved`, and `sub-all-events` has no event-type filter.

### Check code placement and indentation

- Ensure all three implementations are in `client/event_grid_functions.py` between the correct `BEGIN / END` markers.
- Use spaces consistently.
- Preserve the existing imports, constants, client helpers, and subscription names.
- Do not edit `client/app.py`.

### Verify environment variables

- Confirm `.env` and `.env.ps1` exist in the project root.
- Verify they contain `EVENTGRID_ENDPOINT`, `EVENTGRID_TOPIC_NAME`, `RESOURCE_GROUP`, and `NAMESPACE_NAME`.
- From the project root, load variables with `. .\.env.ps1`.
- From the `client` directory, load variables with `. ..\.env.ps1`.
- Each new terminal session needs environment variables loaded separately.

### Check authentication

Verify the Azure CLI session:

```powershell
az account show
```

- Confirm **EventGrid Data Sender** and **EventGrid Data Receiver** roles are assigned on the namespace.
- If necessary, run `python azdeploy.py` and select option **`3`** to assign the roles again.
- Role assignments can take time to propagate; wait briefly and retry.

### Check Python environment and packages

- Ensure `(.venv)` appears in the terminal prompt.
- From `client`, install dependencies with `pip install -r requirements.txt`.
- Check the installed packages with `pip list`.

### Events are redelivered or appear unexpectedly

- Events that are not acknowledged can be redelivered after the receive lock expires.
- The exercise configures a **60-second** receive lock.
- Acknowledged events are removed from the subscription; subsequent receive calls will not return those same acknowledged events.
- The all-events subscription has a separate copy of each published event; acknowledging an event in one subscription does not acknowledge it in another.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure sign-in completed and `Microsoft.EventGrid` registered.
- [ ] Event Grid CLI extension installed.
- [ ] Event Grid Namespace and `moderation-events` topic deployed.
- [ ] `sub-flagged`, `sub-approved`, and `sub-all-events` created.
- [ ] EventGrid Data Sender and EventGrid Data Receiver roles assigned.
- [ ] `.env` and `.env.ps1` generated and environment variables loaded.
- [ ] `publish_moderation_events()` implemented.
- [ ] `check_filtered_delivery()` implemented with acknowledgments for all three subscriptions.
- [ ] `inspect_and_reject()` implemented.
- [ ] Python virtual environment activated and dependencies installed.
- [ ] Flask started and five events published.
- [ ] Expected counts verified: 2 flagged, 2 approved, 5 all-events.
- [ ] Test event inspected and rejected.
- [ ] Flask stopped and Azure resources deleted.

## Key takeaways

- Event Grid namespace topics use CloudEvents v1.0 in this lab.
- Pull delivery lets a consumer explicitly request events rather than relying on an HTTP push endpoint.
- Event subscriptions can filter by event type so downstream handlers receive only relevant events.
- Each subscription maintains its own delivery state; acknowledging an event in one subscription does not acknowledge it in another.
- `acknowledge()` confirms successful processing; `reject()` signals failed processing and can lead to discarding or dead-lettering.
- Microsoft Entra roles allow publishing and receiving events without embedding credentials in the application.
