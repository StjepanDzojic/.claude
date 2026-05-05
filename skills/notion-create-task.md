---
name: notion-create-task
description: Create a Notion task under a specific project. Searches for the project inside the General > Projects section of Notion, verifies it exists, then creates a task. Trigger on: "create a notion task", "add a task in notion", "notion task for <project>".
---

# Notion Create Task

Use this skill whenever the user wants to create a task in Notion.

## Workflow

### 1. Gather task details

Before touching Notion, collect what you need. If the user's message is missing any of the following, use `AskUserQuestion` to ask — one question max, combine into a single call if multiple fields are missing:

- **Project name** — which project the task belongs to
- **Task title** — required
- **Description** — optional
- **Assignee** — optional
- **Due date** — optional
- **Status / priority** — optional (only ask if the database schema supports it)

### 2. Locate the project in Notion

Search Notion for the project using `notion-search`. Start narrow, widen only if needed:

```
query: "<project name>"
query_type: "internal"
page_size: 5
```

Look through the results for a page whose title matches the project name and whose breadcrumb or location suggests it lives under **General > Projects**. The URL or parent chain is the reliable signal — not just the title.

If no result looks right, try a broader search (e.g. just a keyword from the project name). If still nothing matches, **stop and tell the user**:

> "I couldn't find a project called '[name]' under General > Projects in Notion. Please confirm the project name or create it first."

Do not create the task in an unrelated page.

### 3. Find the Tasks database inside the project

Once you have the project page URL/ID, fetch it:

```
notion-fetch: { id: "<project page URL or ID>" }
```

Scan the fetched content for a database or linked database that represents tasks. It may be labeled "Tasks", "To-do", "Action items", or similar. Extract its data source URL (`collection://...`) from the `<data-source url="...">` tag.

If the project page has no tasks database, report this to the user and stop — do not create a loose page as a substitute without asking.

### 4. Fetch the tasks database schema

Fetch the data source to get the exact property names and allowed values:

```
notion-fetch: { id: "<collection://... URL>" }
```

This gives you the `<database>` schema. Read it carefully — field names vary (e.g. "Name" vs "Task" vs "Title"; "Due" vs "Due date"). Use the schema's exact property names when creating the task.

### 5. Create the task

Call `notion-create-pages` with:

- `parent.data_source_id`: the data source ID from step 3
- `pages[0].properties`: mapped from user input to the schema's property names

Only include properties the schema actually has. Do not guess or invent property names.

Example (adapt to real schema):

```json
{
  "parent": { "type": "data_source_id", "data_source_id": "<id>" },
  "pages": [{
    "properties": {
      "Name": "Fix login redirect bug",
      "Status": "To do",
      "Assignee": "Stjepan Džojić",
      "Due date": "2026-05-10"
    }
  }]
}
```

### 6. Confirm success

After the page is created, output the Notion URL of the new task as a clickable markdown link:

> Task created: [Fix login redirect bug](https://notion.so/...)

## Error handling

| Situation | Action |
|---|---|
| Project not found in Notion | Stop, tell user, suggest checking the name |
| Project found but no tasks database | Stop, tell user |
| Schema fetched but required field missing | Use null / omit optional fields; warn user |
| Notion API error | Surface the error message verbatim, do not retry silently |

## Notes

- Never create a task in a random page — always verify the project is under General > Projects.
- Match the project name case-insensitively — Notion search is semantic but titles can have capitalization differences.
- If the user says "create a task" without specifying a project, ask before searching.
