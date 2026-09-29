# Security

**This repository is private. Do not make it public.**

It is not a public demo. It is the working export of a connected system, and it contains
things that would let a reader use, read or interfere with that system. This file exists so
that decision is explicit rather than accidental.

---

## What is in here that is sensitive

| Where | What | Why it matters |
|---|---|---|
| `app/aa-gear-admin.html` | Live n8n webhook URL `https://gl00001.oph.st/webhook/ERPMainSite` | It is reachable by anyone who has the URL, and it **writes to the Airtable base**. There is no auth token on it. |
| `app/aa-gear-landing Final.html` | n8n **test** webhook `https://gl00001.oph.st/webhook-test/ErpAILids` | Same risk while the n8n editor is open. See known issue #2. |
| `workflows/*.json` | Airtable base id `appVONWYUf40A1qX5` | Identifies the base; combined with any credential leak, it points straight at the data. |
| `workflows/*.json` | ~150 n8n **credential id** references | These are pointers to credentials stored in the n8n instance, not the secrets themselves. Publishing them still tells an attacker exactly which credentials to target, and pollutes the n8n instance if the JSON is re-imported. |
| `workflows/WF-9 TelegramBotDirector.json` | Director bot's Telegram chat id `967078186` | Personal identifier of the bot owner. |
| `workflows/*.json` | Internal webhook paths and node topology | The working map of the system. |

### What is **not** in here

The scan for raw secret material came back clean. There are **no** API keys, passwords or
tokens committed:

- no Airtable personal access token (the workflows hold credential *references* only)
- no OpenAI / any LLM provider API key
- no Qdrant API key
- no Google OAuth client secret, refresh token or service-account JSON
- no Telegram bot token
- no `.env` file

That is the good news. The bad news is that none of it is enough: the live webhook needs no
credential of its own, so publishing the URL alone is enough to create records in the base.

---

## Recommended public version, if this ever becomes a portfolio repo

Nothing in `app/` or `workflows/` needs to change in order to demonstrate the work. What has to
change is the plumbing:

1. **Rotate, then replace.**
   - Regenerate the Airtable PAT.
   - Revoke and regenerate the two Telegram bot tokens.
   - Replace the live `/webhook/ERPMainSite` URL with a fresh n8n path, and delete the old
     webhook registration.
2. **Point the HTML at placeholders.** Replace the two webhook URLs with
     `https://example.invalid/webhook/...`, and the base id with `appXXXXXXXXXXXXXX`.
3. **Strip the credential references.** Deleting the `"id"` / `"name"` pairs from the
   `credentials` blocks makes the JSON re-importable anywhere without silently re-attaching a
   stranger's credentials. Leave the node shapes intact — the topology is the interesting part.
4. **Redact the chat id** to a placeholder, or drop the owner-check node's value.
5. **Recreate the screenshots** against the sanitised copy, or accept that the screenshots may
   show real data.

Steps 1 and 2 are the ones that matter. The rest is hygiene.

---

## If it ever leaks

- n8n → deactivate WF-13 and the landing-page webhook immediately.
- n8n → **Credentials** → review every credential referenced by `workflows/`, re-save the ones
  whose ids appear there.
- Airtable → rotate the PAT, then check `Tasks` and the activity log for unexpected writes.
- Telegram → `/revoke` on both bots via BotFather.
- GitHub → force-push the sanitised history, or delete the repository and start again. Assume
  the old data was fetched, because a public push is effectively instant.
