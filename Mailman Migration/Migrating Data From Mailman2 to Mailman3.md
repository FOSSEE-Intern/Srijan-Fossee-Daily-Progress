### Prerequisites: Transfer Data

For migration of data from Mailman 2 two important things are necessary, configuration file (`config.pck`) and the archive file (`mylist.mbox`).

1. **Transfer your data** from the old Mailman 2.1 server to your AlmaLinux machine.
2. **Create a dedicated directory** for the data and set the correct ownership:

```bash
sudo mkdir -p /opt/mailman/mm/import_data
sudo chown -R mailman:mailman /opt/mailman/mm/import_data
```

3. **Place your files here:**
	- `/opt/mailman/mm/import_data/config.pck` (List configuration)
	- `/opt/mailman/mm/import_data/mylist.mbox` (Mbox archives)

### Step 1: Import List Configuration to Mailman Core

This step uses the `mailman` command to import the list settings and membership data.

```bash
sudo su mailman
source /opt/mailman/venv/bin/activate
```
2. **Import the list configuration:**    
```venv
mailman import21 [NEW_LIST_NAME] /opt/mailman/mm/import_data/config.pck
```

**Replace `[NEW_LIST_NAME]`** with the short name you want the list to have in Mailman 3 (e.g., `dev-list`).
- This command will create the mailing list in Mailman Core.

### Step 2: Import Archives to HyperKitty

This step imports the Mbox files into the HyperKitty archiver database. The commands are run using the Django management utility (`mailman-web`).

1. **Import the Mbox archives:**

```venv
mailman-web hyperkitty_import -l [LIST_EMAIL]@example.com /opt/mailman/mm/import_data/[LIST_NAME].mbox
```

   - **Replace `[LIST_EMAIL]@example.com`** with the full email address of the list (e.g., `dev-list@example.com`).
  - **Replace `[LIST_NAME].mbox`** with the actual name of your Mbox file.
### Step 3: Rebuild Search Index (Full-Text Search)

If you successfully set up **Xapian** or **Whoosh** for full-text search, you must rebuild the index so the imported archives are searchable.

1. **Rebuild the search index for the imported list:**

```venv
mailman-web update_index_one_list [LIST_EMAIL]@example.com
```
 - **Replace `[LIST_EMAIL]@example.com`** with the full email address of the imported list.

You should now be able to view the imported list and its archives by accessing the Mailman Web UI via your Host-Only IP.