---

## Part 1: How to Backup Wazuh

### Step 1: Safely Stop the Stack

To prevent database or file corruption while copying the data, stop your Wazuh containers:

```powershell
docker compose down

```

### Step 2: Verify Your Volume Names

Run this command to see the exact names of your Wazuh volumes:

```powershell
docker volume ls | Select-String wazuh

```

If you are inside the `single-node` folder, you will see volumes named **`single-node_wazuh-indexer-data`** (your security alerts/database) and **`single-node_wazuh_etc`** (your manager configurations and agent keys).

### Step 3: Run the Backup Commands

Create a backup folder in your current directory, then use a temporary, lightweight container (`alpine`) to compress and save the data to your Windows drive:

```powershell
# 1. Create a local backup folder
mkdir backup

# 2. Back up the Indexer Database (Alerts, logs, dashboards)
docker run --rm -v single-node_wazuh-indexer-data:/data -v "${PWD}/backup:/backup" alpine tar czf /backup/indexer-data.tar.gz -C /data .

# 3. Back up the Manager Configuration (Agent keys, rules, configurations)
docker run --rm -v single-node_wazuh_etc:/data -v "${PWD}/backup:/backup" alpine tar czf /backup/wazuh-etc.tar.gz -C /data .

```

You will now see `indexer-data.tar.gz` and `wazuh-etc.tar.gz` inside your local Windows `backup` folder. Keep these safe! You can spin your containers back up now using `docker compose up -d`.

---

## Part 2: How to Restore Wazuh

If your environment crashes or you need to migrate to a new machine, use these steps to restore your data.

### Step 1: Prepare the Target Environment

If this is a completely fresh system, complete **Steps 1 and 2 from the installation guide** (clone the repo and generate the certificates).

> **Important:** Ensure the stack is stopped (`docker compose down`) before restoring data. Make sure your `backup` folder containing the two `.tar.gz` files is sitting inside your current directory.

### Step 2: Run the Restore Commands

These commands will clear out any blank/new volume data and extract your backed-up archives directly into the Docker volumes:

```powershell
# 1. Restore the Indexer Database
docker run --rm -v single-node_wazuh-indexer-data:/data -v "${PWD}/backup:/backup" alpine sh -c "rm -rf /data/* && tar xzf /backup/indexer-data.tar.gz -C /data"

# 2. Restore the Manager Configuration & Agent Keys
docker run --rm -v single-node_wazuh_etc:/data -v "${PWD}/backup:/backup" alpine sh -c "rm -rf /data/* && tar xzf /backup/wazuh-etc.tar.gz -C /data"

```

### Step 3: Start the Restored Stack

Bring the stack back online:

```powershell
docker compose up -d

```

Give the indexer a minute or two to initialize. When you log back into the Wazuh Dashboard, all your historical alerts, settings, and connected agents will be exactly as you left them!
