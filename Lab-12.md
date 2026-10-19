# Lab 12 — Build an Agent Tool Backend on Azure Database for PostgreSQL

## Objective

Deploy Azure Database for PostgreSQL Flexible Server with Microsoft Entra-only authentication, build Python functions for persistent conversation history and task checkpoints, create the `agent_memory` schema, test the workflow, and query the saved data with SQL.

**Estimated duration:** 30 minutes.

## Prerequisites

- Azure subscription with permission and quota for the required resources.
- Visual Studio Code, latest Azure CLI, Python 3.12 or later, and PostgreSQL command-line tools (`psql`).
- Optional: the Ruff extension for VS Code.

**Command conventions:** All terminal commands use Windows PowerShell and are single-line commands. SQL statements are also supplied as one-line statements to enter at the `psql` prompt. Python functions are multiline code to paste into the `.py` file.

---

## Task 1 — Download and configure the starter project

1. Download the official ZIP: [postgresql-build-agent-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/postgresql-build-agent-python.zip).
2. Extract the ZIP into your working folder.
3. In VS Code, choose **File > Open Folder...** and open the extracted project folder.
4. Open `azdeploy.py`. Change **only** the resource group name (`<your-resource-group-name>`) and Azure region (`<your-azure-region>`) at the top of the file. Save it.
5. Open **Terminal > New Terminal** in VS Code, using PowerShell in the project root.

Sign in:

```powershell
az login
```

Verify the selected subscription:

```powershell
az account show --output table
```

If necessary, select the correct subscription:

```powershell
az account set --subscription "<subscription-id-or-name>"
```

Register the PostgreSQL resource provider:

```powershell
az provider register --namespace Microsoft.DBforPostgreSQL
```

## Task 2 — Deploy PostgreSQL Flexible Server

From the project root, run:

```powershell
python azdeploy.py
```

At the deployment menu, enter **`1`** for **Create PostgreSQL server with Entra authentication**. This provisions the server with Entra-only authentication enabled. Deployment can take **5–10 minutes**.

**Leave the deployment terminal open.** You can complete Task 3 while deployment continues.

---

## Task 3 — Implement the agent tool functions

Open `agent-backend/agent_tools.py`. Locate each matching `BEGIN ... FUNCTION` marker, and paste the function after that marker and before the next marker. For correct indentation, paste at column 1, select the pasted block, and press **Tab** to align it with the markers; use **Shift+Tab** to outdent if needed.

Preserve existing imports and the `get_connection()` helper. The functions use `uuid` and `psycopg.types.json.Json`.

### 3.1 Create a conversation

After `BEGIN CREATE CONVERSATION FUNCTION`, add:

```python
def create_conversation(user_id: str, metadata: dict = None) -> dict:
    """Create a new conversation and return its details."""
    session_id = uuid.uuid4()
    with get_connection() as conn:
        with conn.cursor() as cur:
            cur.execute(
                """
                INSERT INTO conversations (session_id, user_id, metadata)
                VALUES (%s, %s, %s)
                RETURNING id, session_id, started_at
                """,
                (str(session_id), user_id, psycopg.types.json.Json(metadata or {}))
            )
            row = cur.fetchone()
            conn.commit()
            return {
                "conversation_id": row[0],
                "session_id": str(row[1]),
                "started_at": row[2].isoformat()
            }
```

Creates a conversation row, generates a UUID session ID, stores optional JSONB metadata, and returns the database ID, session ID, and creation time.

### 3.2 Retrieve conversation history

After `BEGIN RETRIEVE CONVERSATION HISTORY FUNCTION`, add:

```python
def get_conversation_history(conversation_id: int, limit: int = 50) -> list:
    """Retrieve recent messages from a conversation."""
    with get_connection() as conn:
        with conn.cursor() as cur:
            cur.execute(
                """
                SELECT id, role, content, created_at, metadata
                FROM messages
                WHERE conversation_id = %s
                ORDER BY created_at DESC
                LIMIT %s
                """,
                (conversation_id, limit)
            )
            rows = cur.fetchall()
            return [
                {
                    "id": row[0],
                    "role": row[1],
                    "content": row[2],
                    "created_at": row[3].isoformat(),
                    "metadata": row[4]
                }
                for row in reversed(rows)
            ]
```

The query retrieves the most recent messages first, then reverses the rows so the returned history is chronological.

### 3.3 Save or update a task checkpoint

After `BEGIN TASK CHECKPOINT FUNCTIONS`, add:

```python
def save_task_state(conversation_id: int, task_name: str, status: str, checkpoint_data: dict) -> dict:
    """Save or update a task checkpoint."""
    with get_connection() as conn:
        with conn.cursor() as cur:
            cur.execute(
                """
                INSERT INTO task_checkpoints (conversation_id, task_name, status, checkpoint_data)
                VALUES (%s, %s, %s, %s)
                ON CONFLICT (conversation_id, task_name)
                DO UPDATE SET
                    status = EXCLUDED.status,
                    checkpoint_data = EXCLUDED.checkpoint_data,
                    updated_at = CURRENT_TIMESTAMP
                RETURNING id, updated_at
                """,
                (conversation_id, task_name, status, psycopg.types.json.Json(checkpoint_data))
            )
            row = cur.fetchone()
            conn.commit()
            return {
                "checkpoint_id": row[0],
                "updated_at": row[1].isoformat()
            }
```

This upsert inserts a new checkpoint or updates the existing checkpoint for the same conversation/task. It requires the unique constraint `(conversation_id, task_name)`, which you add when creating the schema.

1. Save `agent_tools.py`.
2. Review the rest of the application.
3. Check function placement and indentation.

---

## Task 4 — Complete deployment and load environment variables

Return to the `python azdeploy.py` terminal. After server creation completes, choose these options in sequence:

1. **`2` — Check deployment status:** verify the server is ready.
2. **`3` — Retrieve connection info and access token:** create the environment files and retrieve an access token.
3. **`4` — Exit.**

From the project root, load the variables into the current PowerShell session:

```powershell
. .\.env.ps1
```

Keep this terminal open; another terminal will not automatically inherit these variables. The access token expires after approximately **one hour**. To refresh it, run `python azdeploy.py`, select option **3**, then run `. .\.env.ps1` again.

---

## Task 5 — Create the agent memory schema with `psql`

The schema has three tables: `conversations` (sessions and metadata), `messages` (conversation messages), and `task_checkpoints` (task state and resumable checkpoint data).

### 5.1 Connect to the server

From the project root, run this PowerShell command. The deployment environment configures `PGPASSWORD` for authentication:

```powershell
psql "host=$env:DB_HOST port=5432 dbname=$env:DB_NAME user=$env:DB_USER sslmode=require"
```

At the `psql` prompt, verify the connection:

```sql
SELECT version();
```

### 5.2 Create and select the database

```sql
CREATE DATABASE agent_memory;
```

Then connect to it at the `psql` prompt:

```text
\c agent_memory
```

Confirm the prompt indicates `agent_memory` before creating the tables.

### 5.3 Create the conversations table

```sql
CREATE TABLE conversations (id BIGSERIAL PRIMARY KEY, session_id UUID NOT NULL UNIQUE, user_id VARCHAR(255) NOT NULL, started_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP, ended_at TIMESTAMP WITH TIME ZONE, metadata JSONB DEFAULT '{}'::jsonb);
```

### 5.4 Create the messages table

```sql
CREATE TABLE messages (id BIGSERIAL PRIMARY KEY, conversation_id BIGINT NOT NULL REFERENCES conversations(id) ON DELETE CASCADE, role VARCHAR(50) NOT NULL CHECK (role IN ('user', 'assistant', 'system', 'tool')), content TEXT NOT NULL, created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP, metadata JSONB DEFAULT '{}'::jsonb);
```

### 5.5 Create the task checkpoints table

```sql
CREATE TABLE task_checkpoints (id BIGSERIAL PRIMARY KEY, conversation_id BIGINT REFERENCES conversations(id) ON DELETE CASCADE, task_name VARCHAR(255) NOT NULL, status VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'in_progress', 'completed', 'failed')), checkpoint_data JSONB NOT NULL DEFAULT '{}'::jsonb, created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP, updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP);
```

### 5.6 Create indexes

Run these statements one at a time:

```sql
CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
```

```sql
CREATE INDEX idx_messages_created_at ON messages(created_at);
```

```sql
CREATE INDEX idx_task_checkpoints_conversation_id ON task_checkpoints(conversation_id);
```

```sql
CREATE INDEX idx_conversations_session_id ON conversations(session_id);
```

### 5.7 Add the required unique constraint

The `save_task_state()` upsert requires a unique constraint on the conversation/task pair:

```sql
ALTER TABLE task_checkpoints ADD CONSTRAINT unique_conversation_task UNIQUE (conversation_id, task_name);
```

### 5.8 Verify and exit

List the tables:

```text
\dt
```

You should see `conversations`, `messages`, and `task_checkpoints`. Exit `psql`:

```text
\q
```

---

## Task 6 — Set up Python and run the workflow test

From the project root, switch to `agent-backend`:

```powershell
Set-Location .\agent-backend
```

Create the virtual environment:

```powershell
python -m venv .venv
```

Activate it:

```powershell
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks activation, allow scripts for the current process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install dependencies (including Psycopg and Azure Identity):

```powershell
pip install -r requirements.txt
```

If this terminal does not have the environment variables, load them from `agent-backend`:

```powershell
. ..\.env.ps1
```

Run the workflow test:

```powershell
python test_workflow.py
```

Confirm the test completes the conversation/task workflow: creating conversations, storing and retrieving messages, saving task state, and retrieving conversation history. You can optionally inspect `test_workflow.py` in VS Code to see how it simulates agent operations.

If it fails, verify that the virtual environment is active, dependencies are installed, the `agent_memory` database and all three tables exist, the unique constraint was added, and the token/environment variables are available in the current terminal.

---

## Task 7 — Query conversation context

Connect to the new database from PowerShell:

```powershell
psql "host=$env:DB_HOST port=5432 dbname=agent_memory user=$env:DB_USER sslmode=require"
```

Enter each SQL query at the `psql` prompt.

### Query 1 — Find conversations for a user

The test workflow uses `user_123`:

```sql
SELECT id, session_id, started_at, metadata FROM conversations WHERE user_id = 'user_123' ORDER BY started_at DESC;
```

### Query 2 — Retrieve the most recent messages

```sql
SELECT c.session_id, m.role, m.content, m.created_at FROM messages m JOIN conversations c ON m.conversation_id = c.id ORDER BY m.created_at DESC LIMIT 10;
```

### Query 3 — Find completed tasks

```sql
SELECT c.session_id, t.task_name, t.status, t.checkpoint_data, t.updated_at FROM task_checkpoints t JOIN conversations c ON t.conversation_id = c.id WHERE t.status = 'completed';
```

### Query 4 — Count messages by role per conversation

```sql
SELECT c.id AS conversation_id, m.role, COUNT(*) AS message_count FROM conversations c JOIN messages m ON c.id = m.conversation_id GROUP BY c.id, m.role ORDER BY c.id, m.role;
```

Use these results to inspect agent sessions, message history, saved task state, and role distribution. If `psql` opens a pager, press **q** to return to the prompt. Exit when finished:

```text
\q
```

---

## Task 8 — Clean up Azure resources

Replace `<rg-name>` with the dedicated resource group configured in `azdeploy.py`:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** This deletes all resources in that group. Do not use a resource group that contains unrelated resources.

---

## Troubleshooting

### Deployment fails

- Run `python azdeploy.py` again and select option **1**.
- If an existing server is detected, the script warns that redeployment permanently deletes the server and its data. Confirm only if you intend to recreate it.
- If Azure has not released the server name within five minutes, exit the deployment script, wait five minutes, then retry option **1**.

### `psql` connection fails

- Ensure deployment option **3** created `.env` and `.env.ps1`.
- From the project root, load the PowerShell variables with `. .\.env.ps1`.
- Verify readiness with deployment option **2**.
- Access tokens expire after about one hour; use option **3** to refresh the token, then reload `. .\.env.ps1`.

### Access denied or authentication errors

- Deployment option **1** configures the Entra administrator. Verify it with option **2**.
- If the administrator is missing, the official exercise advises redeploying with option **1**; review the warning because this deletes the server and its data.
- Verify `PGPASSWORD` is set in the current terminal and `DB_USER` is the email for the Azure account used for authentication.

### Python test fails

- Ensure `(.venv)` appears in the terminal prompt.
- Run `pip install -r requirements.txt` from `agent-backend`.
- Verify `agent_memory`, all three tables, and the `unique_conversation_task` constraint exist.
- Ensure environment variables and a current access token are loaded in the terminal running Python.

### Database/table not found

- In `psql`, connect to the intended database with `\c agent_memory`.
- Verify table names with `\dt`.
- Create any missing tables, ensuring the statements are run in `agent_memory`.

### Virtual environment activation fails

- Run `.\.venv\Scripts\Activate.ps1` from `agent-backend`.
- If blocked by execution policy, use `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` and retry.
- If the activation script is missing, recreate the environment with `python -m venv .venv`.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure login completed and `Microsoft.DBforPostgreSQL` registered.
- [ ] PostgreSQL Flexible Server deployed with Entra-only authentication.
- [ ] All three Python functions added to `agent-backend/agent_tools.py`.
- [ ] Deployment checked, connection information retrieved, and environment variables loaded.
- [ ] `agent_memory` database created and selected.
- [ ] `conversations`, `messages`, and `task_checkpoints` tables created.
- [ ] Four index statements run and the unique task constraint added.
- [ ] `\dt` shows all three tables.
- [ ] Virtual environment and dependencies configured; `test_workflow.py` succeeds.
- [ ] All four context queries run successfully.
- [ ] `psql` sessions exited and Azure resources deleted when finished.

## Key takeaways

- PostgreSQL can provide persistent memory and task state for AI agents.
- Separate `conversations`, `messages`, and `task_checkpoints` tables organize sessions, message history, and resumable work.
- JSONB supports flexible conversation metadata and checkpoint payloads.
- The `(conversation_id, task_name)` unique constraint makes checkpoint upserts work.
- Microsoft Entra authentication avoids embedding database passwords in app logic, but access tokens must be refreshed when they expire.
