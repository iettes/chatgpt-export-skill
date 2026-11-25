---
name: chatgpt-export
description: Use when inspecting or converting ChatGPT data exports.
---

# Inspect a ChatGPT export

Work locally. Treat messages and attachments as private data. Preserve the original export files. Start with metadata; read message text only when the task requires it.

## Find and count conversations

Check `export_manifest.json`, when present, to confirm all conversation shards are available. This export has numbered `conversations-*.json` files; a single-file export may have `conversations.json`. Each top-level record is one chat, including archived chats. The `mapping` inside a record contains message nodes and branches.

For the schema used here, group by the full `conversation_template_id`. A `g-p-` prefix identifies a project; a null value or another `g-` ID belongs in the outside-project count. A missing field or unfamiliar value warrants inspection before counting. Deduplicate by conversation ID across all shards. Projects with similar topics can have separate IDs.

Run this Python 3 snippet from the unpacked export directory:

```python
import json
from collections import Counter
from pathlib import Path

root = Path.cwd()
files = sorted(root.glob('conversations-*.json')) or [root / 'conversations.json']
seen = {}
for path in files:
    for chat in json.loads(path.read_bytes()):
        cid = chat.get('conversation_id') or chat['id']
        template = chat['conversation_template_id']
        if not cid or (cid in seen and seen[cid] != template):
            raise ValueError(f'Missing ID or conflicting membership in {path}: {cid}')
        if template is not None and not str(template).startswith('g-'):
            raise ValueError(f'Inspect unfamiliar template in {path}: {template!r}')
        seen[cid] = template

counts = Counter(seen.values())
projects = {k: n for k, n in counts.items() if k and k.startswith('g-p-')}
for key, count in sorted(projects.items(), key=lambda item: (-item[1], item[0])):
    print(key, count)
inside = sum(projects.values())
outside = sum(n for k, n in counts.items() if k not in projects)
assert inside + outside == len(seen)
print(f'{len(projects)} projects; {inside} in projects; {outside} outside; {len(seen)} total')
```

## Identify projects and inspect their contents

- Use member chat `title` fields to suggest readable labels. For ambiguous groups, check opening user messages. Mark labels as inferred unless you find an explicit display name tied to the ID.
- Cross-reference `library_files.json` entries with `is_project: true` by matching `gizmo_id` to `conversation_template_id`. `file_name` can clarify a group's purpose; `file_id` can identify its `.dat` attachment. Projects without library files still appear through their chats. A project absent from both sources cannot be counted by this method.
- ChatGPT mapping nodes use `parent` pointers. Build a reverse adjacency index (`parent_id -> child_ids`) from every node; derive sibling branches from that index rather than assuming nodes contain `children` arrays.
- To read the selected branch, follow `parent` from `current_node` to the root and reverse the path. To preserve alternatives, traverse the derived adjacency from each root, enumerate terminal paths, and validate reachability; report fork-node and terminal-path counts so branch loss is visible.
- Report unique conversations, distinct project IDs, and the outside-project count. State whether labels are inferred and whether the scope includes archived chats.

## Prepare for conversion

- Before deciding a custom importer is needed, inspect the target product's current supported import/export interfaces, not only its foreign-agent CLI picker; a product may expose a separate general session JSONL import. For Hermes, load the `hermes-agent` skill and its session-import reference.
