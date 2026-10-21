# Lab 16 — Publish and Subscribe to Events in Azure Managed Redis

## Lab objective

Deploy Azure Managed Redis and complete a Flask web application that publishes messages to Redis channels and receives them through direct and pattern subscriptions. You will authenticate with Microsoft Entra ID, publish individual events, broadcast to multiple channels, format incoming messages, and listen for events on a background thread.

**Estimated duration:** 30 minutes.

## Prerequisites

- An Azure subscription with permission and quota to create the required resources.
- Visual Studio Code (VS Code).
- Python 3.12 or later.
- Latest Azure CLI.
- Azure CLI `redisenterprise` extension version **2.75.0 or later**.
- Optional: Ruff extension for VS Code.

**Command convention:** All terminal commands in this guide use Windows PowerShell and are single-line commands. The Python functions below are multiline code to paste into `client/pubsub_functions.py`; they are not shell commands.

---

## Task 1 — Download the starter project and start deployment

1. Download the official starter ZIP file:

   [Download amr-pub-sub-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/amr-pub-sub-python.zip)

2. Move the ZIP file to your working folder and extract it.
3. Open VS Code, select **File > Open Folder...**, and open the extracted project folder.
4. Open `azdeploy.py`.
5. Update **only** the two values at the top of the deployment script:
   - Resource group name: replace `<your-resource-group-name>` with a dedicated resource group name.
   - Azure region: replace `<your-azure-region>` with a region available to your subscription.
6. Save `azdeploy.py`. Do not modify anything else in this script.
7. In VS Code, select **Terminal > New Terminal**. Use PowerShell in the project root.

Sign in to Azure:

```powershell
az login
```

Follow the browser prompts and select the correct Azure account and subscription.

Register the Azure Managed Redis resource provider:

```powershell
az provider register --namespace Microsoft.Cache
```

Install or upgrade the Azure CLI extension. Version 2.75.0 or later is required for Microsoft Entra ID data access configuration:

```powershell
az extension add --upgrade --name redisenterprise
```

Start the deployment script:

```powershell
python azdeploy.py
```

When the menu appears, enter **`1`** for **Create Azure Managed Redis resource**. This creates the resource group if it does not exist and deploys Azure Managed Redis. Deployment takes about **5–10 minutes**.

**Leave the deployment script running.** Complete the Python code tasks while Azure provisions the resource. Check the deployment terminal periodically for errors.

---

## Task 2 — Complete the Redis client connection

Open `client/pubsub_functions.py` in VS Code. The file contains `BEGIN / END` markers for each implementation section.

**Indentation tip:** Paste the code flush with the left margin (column 1), select the pasted lines, and press **Tab** to align them with the existing markers. Use **Shift+Tab** to outdent if needed. Use consistent Python indentation. Do not change code outside the designated sections.

Find `# BEGIN CONNECTION CODE SECTION` and add:

```python
def get_client() -> redis.Redis:
    """Create a Redis client for Azure Managed Redis using Microsoft Entra ID."""
    redis_host = os.environ.get("REDIS_HOST")

    if not redis_host:
        raise ValueError("REDIS_HOST environment variable must be set")

    # DefaultAzureCredential acquires a Microsoft Entra token for Redis.
    # The credential provider refreshes the token for long-lived connections.
    credential_provider = create_from_default_azure_credential(
        ("https://redis.azure.com/.default",),
    )

    return redis.Redis(
        host=redis_host,
        port=10000,
        ssl=True,
        decode_responses=True,
        credential_provider=credential_provider,
        socket_timeout=30,
        socket_connect_timeout=30,
    )
```

Key details:

- The function reads the endpoint from the `REDIS_HOST` environment variable.
- Azure Managed Redis uses TLS over port `10000`.
- `decode_responses=True` returns strings rather than byte strings.
- `create_from_default_azure_credential()` uses `DefaultAzureCredential` to acquire and refresh Microsoft Entra tokens, supporting long-lived pub/sub connections.
- If `REDIS_HOST` is missing, the function raises a clear configuration error.

Save the file and review the code.

---

## Task 3 — Add the order-created event publisher

Find `# BEGIN PUBLISH MESSAGE CODE SECTION` and add:

```python
def publish_order_created(r: redis.Redis) -> dict:
    """Publish an order created event to the 'orders:created' channel."""
    order_data = {
        "event": "order_created",
        "order_id": f"ORD-{datetime.now().strftime('%Y%m%d%H%M%S')}",
        "customer": "Jane Doe",
        "total": 129.99,
        "timestamp": datetime.now().isoformat(),
    }
    channel = "orders:created"

    # publish() sends to every subscriber and returns the number reached.
    subscribers = r.publish(channel, json.dumps(order_data))

    return {"channel": channel, "subscribers": subscribers, "message": order_data}
```

This function creates a sample order event, serializes the event as JSON, and publishes it to the `orders:created` channel. The return value from `publish()` is the number of subscribers that received the message at that time.

The starter file already includes these event publishers; review them but do not replace them:

- `publish_order_shipped()`
- `publish_inventory_alert()`
- `publish_notification()`

Save the file.

---

## Task 4 — Add the broadcast-to-all function

Find `# BEGIN BROADCAST CODE SECTION` and add:

```python
def broadcast_to_all(r: redis.Redis) -> dict:
    """Broadcast the same message to every channel using publish() in a loop."""
    announcement = {
        "event": "system_announcement",
        "message": "System maintenance scheduled for 2 AM",
        "priority": "high",
        "timestamp": datetime.now().isoformat(),
    }
    message = json.dumps(announcement)

    results = []
    total_subscribers = 0
    for channel in AVAILABLE_CHANNELS:
        # Send the same message to every known channel.
        count = r.publish(channel, message)
        total_subscribers += count
        results.append({"channel": channel, "subscribers": count})

    return {
        "channels": results,
        "total_subscribers": total_subscribers,
        "message": announcement,
    }
```

This function sends the same maintenance announcement to every channel in `AVAILABLE_CHANNELS` and adds up the subscriber counts. A channel only delivers the message to clients that have a matching active subscription.

Save the file.

---

## Task 5 — Add the received-message formatter

Find `# BEGIN MESSAGE FORMATTING CODE SECTION` and add:

```python
def format_message(message: dict) -> dict:
    """Parse a pub/sub message and extract relevant fields for display."""
    timestamp = datetime.now().strftime("%H:%M:%S")
    channel = message.get("channel", "unknown")
    pattern = message.get("pattern")

    try:
        data = json.loads(message["data"])
    except (json.JSONDecodeError, TypeError):
        # Preserve a non-JSON payload rather than dropping it.
        return {
            "timestamp": timestamp,
            "channel": channel,
            "pattern": pattern,
            "event": None,
            "details": {"raw": message.get("data")},
        }

    field_names = [
        "order_id", "customer", "total", "tracking_number",
        "product_name", "current_stock", "message",
    ]
    details = {name: data[name] for name in field_names if name in data}

    return {
        "timestamp": timestamp,
        "channel": channel,
        "pattern": pattern,
        "event": data.get("event", "unknown"),
        "details": details,
    }
```

The formatter extracts the channel, optional matching pattern, event name, and display fields. If a payload is not valid JSON, it retains the raw content so the message is not lost.

Save the file.

---

## Task 6 — Implement the background message listener

Find `# BEGIN MESSAGE LISTENER CODE SECTION` inside the `PubSubManager` class and add the following method. **This method must be indented as a class method**, one level inside the class.

```python
def listen_messages(self) -> None:
    """Background thread that reads messages from subscribed channels."""
    self.listener_active = True
    try:
        # listen() blocks and yields messages as they arrive.
        for message in self.pubsub.listen():
            if not self.listening:
                break

            # Direct subscriptions yield "message"; pattern subscriptions yield "pmessage".
            if message["type"] in ("message", "pmessage"):
                self._add_message(format_message(message))

    except Exception as e:
        if self.listening:
            self._add_message({
                "timestamp": datetime.now().strftime("%H:%M:%S"),
                "channel": "system",
                "pattern": None,
                "event": "listener_error",
                "details": {"error": str(e)},
            })
    finally:
        self.listener_active = False
```

This method runs on a background thread. It continuously listens for messages, handles both direct-channel messages (`message`) and pattern-subscription messages (`pmessage`), formats them, and adds them to the app's thread-safe message buffer. If an error occurs while listening, it records a system message.

Save the file and verify the method remains inside the `PubSubManager` class.

---

## Task 7 — Add channel and pattern subscriptions

Find `# BEGIN SUBSCRIBE CHANNEL/PATTERN CODE SECTION` inside the `PubSubManager` class. Add both methods at the class-method indentation level:

```python
def subscribe_to_channel(self, channel: str) -> str:
    """Subscribe to a specific channel using subscribe()."""
    self.pubsub.subscribe(channel)
    self.restart_listener()
    return f"Subscribed to channel: {channel}"

def subscribe_to_pattern(self, pattern: str) -> str:
    """Subscribe using a pattern with psubscribe() (for example, 'orders:*')."""
    self.pubsub.psubscribe(pattern)
    self.restart_listener()
    return f"Subscribed to pattern: {pattern}"
```

- `subscribe()` receives messages from one exact channel.
- `psubscribe()` receives messages from channels matching a wildcard pattern, such as `orders:*`.
- `restart_listener()` restarts the background listener after a subscription change so it can consume messages from the updated subscription set.

Save `pubsub_functions.py`.

**Do not edit `client/app.py`.** The starter Flask app already calls the functions you implemented, provides the publish and subscription interface, and streams received messages to the browser.

---

## Task 8 — Finish deployment and configure access

Return to the terminal running `python azdeploy.py`. After the resource has been created successfully, press **Enter** to return to the deployment menu. Then choose these options in order:

1. Enter **`2`** — **Create database and configure access**. This creates the database, assigns a Microsoft Entra ID data access policy to your account, and creates `.env` and `.env.ps1` files containing the Redis endpoint (`REDIS_HOST`).
2. Enter **`3`** — **Check deployment status**. Confirm the cluster and database are ready.
3. Enter **`4`** — **Exit**.

Load the PowerShell environment variables from the project root:

```powershell
. .\.env.ps1
```

Keep this terminal open. If you open a new terminal, load the environment variables again in that session.

---

## Task 9 — Create the Python environment

From the project root, navigate to the `client` folder:

```powershell
Set-Location .\client
```

Create the virtual environment:

```powershell
python -m venv .venv
```

Activate it in Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks activation because of execution policy, allow it for the current process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the application dependencies:

```powershell
pip install -r requirements.txt
```

Check the installed packages if needed:

```powershell
pip list
```

The prompt should show `(.venv)` when the virtual environment is active.

If this terminal was opened after loading the variables in another terminal, load the environment from the project root by running the following from `client`:

```powershell
. ..\.env.ps1
```

---

## Task 10 — Run the Flask application

Make sure you are in `client`, the virtual environment is active, and environment variables are loaded. Start the app:

```powershell
python app.py
```

Leave the application running. Open a browser at:

[http://localhost:5000](http://localhost:5000)

The app displays a left panel for publishing events and managing subscriptions, and a right panel showing the last publish result and live received messages.

### 10.1 Subscribe to notifications

1. In **Subscriptions** on the left panel, open the channel dropdown.
2. Choose **`notifications`** and select **Subscribe**.
3. Confirm a success message appears and `notifications` is shown in **Active Subscriptions**.

You must subscribe before publishing if you want to receive a channel's messages.

### 10.2 Publish a notification

1. In **Publish Events**, select **Notification**.
2. Review the publish result in the right panel. It should show the channel and the number of subscribers reached.
3. Within a second or two, confirm the message appears under **Received Messages**.

The message appears because the app is subscribed to the `notifications` channel and its background listener is active.

### 10.3 Publish an inventory alert

1. Select **Inventory Alert**.
2. Confirm that the publish result indicates a message was sent to `inventory:alerts`.
3. Observe that the message does **not** appear in **Received Messages**, because you subscribed only to `notifications`.

This demonstrates that Redis pub/sub does not deliver a message to clients that have no matching subscription.

### 10.4 Subscribe to an orders pattern

1. In **Subscriptions**, enter `orders:*` in the pattern field.
2. Select **Subscribe to Pattern**.
3. Confirm the pattern appears in **Active Subscriptions** alongside `notifications`.

The pattern matches channels that begin with `orders:`.

### 10.5 Publish order events

1. Select **Order Created**.
2. Confirm the message arrives in **Received Messages**.
3. Select **Order Shipped**.
4. Confirm that message also arrives.

Both messages are delivered because the `orders:*` pattern matches `orders:created` and `orders:shipped`.

### 10.6 Broadcast to all channels

1. Select **Broadcast to All**.
2. Watch **Received Messages** update for the announcement messages that match the active subscriptions.
3. Review the publish result, including the per-channel subscriber counts.

The function publishes the same announcement to every channel in `AVAILABLE_CHANNELS`. A message appears only if one of your channel or pattern subscriptions matches its channel.

### 10.7 Unsubscribe and test again

1. Select **Unsubscribe from All**.
2. Select **Broadcast to All** again.
3. Confirm no new messages arrive in **Received Messages**, because there are no active subscriptions.

When you finish testing, return to the terminal running Flask and press **Ctrl+C** to stop the application.

---

## Task 11 — Clean up Azure resources

Delete the dedicated resource group configured in `azdeploy.py`. Replace `<rg-name>` with the actual resource group name:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting the resource group deletes all resources inside it. Use a dedicated resource group. If you chose an existing group that contains unrelated resources, those will also be deleted.

---

## Troubleshooting

### Verify Azure Managed Redis deployment

- In the Azure portal, open the resource group and confirm the Azure Managed Redis resource has **Provisioning State: Succeeded**.
- Run the deployment script's **Check deployment status** option and confirm the cluster and database are ready.
- If status is not `Succeeded`, wait and run option **3** again.

### Deployment fails because of regional capacity

- Capacity for a chosen SKU can be temporarily unavailable.
- Exit the deployment script and change only the Azure region variable near the top of `azdeploy.py` to another available region, such as `eastus2`, `australiaeast`, or `canadacentral`.
- Save the file, rerun `python azdeploy.py`, and select option **1**. The script deletes the failed resource automatically before another attempt.

### Authentication or access errors

- Confirm Azure CLI sign-in:

```powershell
az account show
```

- Ensure deployment menu option **2** completed successfully and configured the Microsoft Entra ID data access policy.
- If access was just assigned, wait briefly and retry because policy assignment can take time to propagate.

### Code errors or indentation issues

- Confirm all code blocks are in the correct sections of `client/pubsub_functions.py`.
- Use spaces consistently.
- Ensure `listen_messages()`, `subscribe_to_channel()`, and `subscribe_to_pattern()` are indented as methods inside `PubSubManager`.
- Do not remove or modify code outside the designated markers.

### Environment variables are missing

- Check that `.env` and `.env.ps1` exist in the project root and contain `REDIS_HOST`.
- From the project root, run `. .\.env.ps1`.
- From `client`, run `. ..\.env.ps1`.
- Each new terminal session needs the variables loaded separately.

### No messages appear

- Confirm you subscribed to the exact channel being published, or to a matching pattern.
- Check **Active Subscriptions** in the browser.
- Confirm the Flask app is still running and the message stream is updating.
- Remember: messages published before a subscription is active are not replayed later. Redis Pub/Sub is a live messaging mechanism, not a persistent message queue.

### Python environment or package errors

- Confirm `(.venv)` appears in the prompt.
- Run `pip install -r requirements.txt` from `client`.
- Check installed packages using `pip list`.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure sign-in completed and `Microsoft.Cache` registered.
- [ ] `redisenterprise` extension installed or upgraded to version 2.75.0 or later.
- [ ] Azure Managed Redis deployed successfully.
- [ ] `get_client()` implemented with Microsoft Entra ID, TLS, and port 10000.
- [ ] `publish_order_created()` and `broadcast_to_all()` implemented.
- [ ] `format_message()` and the background `listen_messages()` method implemented.
- [ ] Direct channel and pattern subscription methods implemented.
- [ ] Database/access setup completed; `.env` and `.env.ps1` created.
- [ ] Python environment activated and dependencies installed.
- [ ] Notification subscription tested.
- [ ] Inventory alert tested while unsubscribed from its channel.
- [ ] `orders:*` pattern tested with order-created and order-shipped events.
- [ ] Broadcast and unsubscribe behavior tested.
- [ ] Flask stopped and Azure resources deleted.

## Key takeaways

- Redis Pub/Sub is a live messaging pattern: publishers send messages to channels and Redis forwards them to current subscribers.
- `publish()` returns the number of subscribers that received a message.
- `subscribe()` listens to one exact channel; `psubscribe()` listens to channel patterns such as `orders:*`.
- A background listener lets the Flask app receive events while still responding to web requests.
- Pub/Sub messages are not stored for later delivery; a consumer that subscribes after a message is published will not receive that earlier message.
- Azure Managed Redis can authenticate with Microsoft Entra ID using `DefaultAzureCredential` and TLS on port `10000`.
