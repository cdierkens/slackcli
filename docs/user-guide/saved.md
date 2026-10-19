# Work through Slack saved items

`slackcli saved list` reads your **Later** / "saved for later" list;
`add`, `update`, `complete`, `reopen` and `remove` change it.

## `saved list`

```bash
slackcli saved list
slackcli saved list --limit=50
slackcli saved list --state=to_do
slackcli saved list --json
```

| Option | Purpose |
|---|---|
| `--limit <number>` | Maximum number of items to return |
| `--state <state>` | Filter by `saved`, `to_do`, or `completed` |
| `--workspace <id\|name>` | Workspace to use |
| `--json` | JSON output |
| `--fields <list>` | With `--json`, only these comma-separated fields (dot paths allowed) |

SlackCLI pages through the whole list (or stops at `--limit`). `--limit` caps
the items fetched, before `--state` filters them.

The two auth types read different Slack lists and return different shapes:

- **Browser auth** uses Slack's `saved.list`, the **Later** list. Raw saved
  entries are only pointers — a channel ID and a timestamp — so SlackCLI
  resolves each one into the message text, the channel name, and the author.
  Each `--json` item is `{type, channel_id, channel_name, message, date_saved,
  todo_state}`; non-message items carry only `type`, `channel_id` and
  `date_saved`.
- **Standard auth** (app tokens) falls back to `stars.list`, which returns
  *starred* items with the message inline. They are passed through as Slack
  returns them, so the `--json` item shape is Slack's `stars.list` shape. Those
  items have no `todo_state`, so `--state` matches nothing.

`--json` emits `{item_count, items}`. When there are no items, nothing is
written to stdout and the command exits 0.

## Changing the Later list

`saved add`, `update`, `complete`, `reopen` and `remove` act on **one message**
on your Later list. **Browser authentication only**: they use the
`saved.add`, `saved.update` and `saved.delete` endpoints of Slack's web client,
which authenticate with a browser session; an app token is refused with
`unsupported_auth_type` (its `stars.*` list is a different list).

Name the message the way `messages delete` does: `--channel-id` (an ID, a Slack
URL or a channel name) and `--timestamp`, or a single `--permalink`. See
[links and timestamps](links-and-timestamps.md).

```bash
slackcli saved add --permalink="$LINK"                                  # a to-do, due now
slackcli saved add --channel-id=C1234567890 --timestamp=1712345678.123456 --due "2026-10-12 09:50"
slackcli saved update --permalink="$LINK" --due 2026-10-12T09:50:00+02:00
slackcli saved update --permalink="$LINK" --no-due
slackcli saved update --permalink="$LINK" --state saved                 # move to Saved
slackcli saved complete --permalink="$LINK"
slackcli saved reopen --permalink="$LINK"                               # back to To-dos, due now
slackcli saved remove --permalink="$LINK" --yes
```

| Command | What it sends |
|---|---|
| `saved add [--due <time>]` | Adds the message as a **to-do**, due at `--due`, or now (as Slack's web client does when no date is picked) |
| `saved update --due <time>` / `--no-due` | Changes or clears the due date only |
| `saved update --state saved` | Moves the item to **Saved** and clears its due date (takes no `--due`/`--no-due`) |
| `saved update --state to_do [--due <time>]` | Moves the item to **To-dos**, due at `--due` or now (takes no `--no-due`) |
| `saved complete` | Marks the item completed and clears its due date |
| `saved reopen` | Marks a completed item not completed: back to **To-dos**, due now |
| `saved remove` | Removes the item from the Later list; the message itself is not touched |

`--due` takes the same formats as `messages schedule --at`: Unix seconds
(`1791791400`), ISO 8601 with an offset (`2026-10-12T09:50:00+02:00` or `...Z`),
or a local date and time (`"2026-10-12 09:50"`). Unlike `--at` it may be in the
past (an overdue to-do) and has no 120-day limit.

| Option | Purpose |
|---|---|
| `--channel-id <id>` | Channel ID, URL or name where the message is |
| `--timestamp <ts>` | Message timestamp (`1712345678.123456` or `p1712345678123456`) |
| `--permalink <url>` | Slack message link (replaces `--channel-id` and `--timestamp`) |
| `--due <time>` | (`add`, `update`) Due date and time |
| `--no-due` | (`update`) Clear the due date |
| `--state <state>` | (`update`) `saved` or `to_do` |
| `--yes` | (`remove`) Confirm without a prompt |
| `--workspace <id\|name>` | Workspace to use |
| `--json` | JSON output |
| `--dry-run` | Show what would be sent, without sending it |

`add`, `update`, `complete` and `reopen` act immediately. `remove` asks for
confirmation on a terminal and needs `--yes` when run unattended.

With `--json`, `add`, `update`, `complete` and `reopen` print the item as Slack
reports it after the change: `{channel_id, ts, todo_state, date_due}`
(`date_due` in Unix seconds, `0` for none). `remove` prints
`{channel_id, ts, removed: true}`. A Slack refusal exits 1 with the Slack code
in `slack_error`.

Only single-item changes are offered: SlackCLI never calls Slack's bulk Later
methods, some of which act on the whole list.
