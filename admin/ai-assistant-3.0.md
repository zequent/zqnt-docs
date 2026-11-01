# AI assistant: files and profiles (3.0 preview)

How an organization admin sets up two parts of the AI assistant in 3.0: **files in the chat** and
**assistant profiles**. This page describes a preview; details can change until 3.0 is released.

## Files in the chat

People can attach files and photos to a conversation, for example a manual, a spreadsheet of
inspection points or a KML of a site, and ask the assistant about them. Every file is checked by
its content and scanned for viruses before anything reads it. The assistant reads files in an
isolated sandbox without network access.

### Switching it on

Files are off until an organization admin switches them on. In the console:

1. **Manage → AI Assistant → Models & budget → What the model may see:** switch on **Content of
   files people attach to a chat**. The text of a file goes to the model provider with the question,
   so without this switch nobody can attach a file.
2. **Files in chat** (same tab): switch on **People may attach files**, and choose:
   - the **file types**: PDF, Word, Excel, CSV, KML, GeoJSON, Text, PNG and JPEG (none ticked means
     all of them),
   - **Largest file** (MB), **Files per conversation** and **Storage for the organization** (MB).
     0 means the platform's limit; a higher value than the platform allows is capped to it.
3. **Roles** tab: switch on **May attach files to a chat** for the roles that may use it.

The platform's own limits are 25 MB per file, 20 files per conversation and 1 GB of storage per
organization, unless the operator set others.

### Good to know

- **Photos** (PNG, JPEG) are only offered while a model the person may use can see images. They are
  re-encoded before a model sees them, so their metadata, including GPS positions, never reaches a
  model.
- A file that the scan flags is refused and kept in quarantine; the assistant never reads it.
- Files are deleted with their conversation.

### For operators (Helm chart)

Uploads need the virus scanner. Without it, or while it is down, every upload is refused.

| Value | What to set |
| --- | --- |
| `fileScanner.enabled` | `true`. Runs ClamAV (about 1.5 GB of memory, 2 Gi of storage for its signatures). It downloads its signatures from `database.clamav.net` (port 443) and keeps them current; uploads are refused until the first download is done, which takes a few minutes |
| `objectStorage.filesBucket`, `objectStorage.quarantineBucket` | the buckets for clean and flagged files (default `zqnt-files` and `zqnt-quarantine`), created by the chart's storage job on install or upgrade |
| `agentService.resources.limits.memory` | `2Gi` or more: reading a large spreadsheet or PDF can take up to 1 GB, two at a time |

The console's ingress accepts uploads up to 30 MB. The Docker Compose setup has no virus scanner
yet, so files cannot be attached there.

## Assistant profiles

A profile tells the assistant how to work for your organization or a site: a role, house rules,
which Skills and tools it may use, which model it uses. Open **Manage → Agent Profiles** and create
one:

| Field | Meaning |
| --- | --- |
| **Enabled** | only enabled profiles are used |
| **Default for its scope** | a conversation that asks for no particular profile gets the default of its site, else the organization's default |
| **Applies to** | the organization, or one site |
| **Instructions** | role and instructions on top of the platform's own, up to 20,000 characters. Good for knowledge the assistant cannot look up: which drone answers which alarm, the names of your plant's signals, how your demo is set up |
| **Skills it may use**, **Assistant tools** | narrow what the assistant may do. Left empty means no restriction |
| **What it knows about** | sites or assets the profile is meant for; the assistant looks them up when a question is about them |
| **Model**, **Thinking** | optional; empty uses the organization's settings |

Instructions can narrow what the assistant does and add context. They never give it more rights
than the person it works for has, and they do not switch off approvals. Start a new conversation
after changing a profile to be sure the change applies.
