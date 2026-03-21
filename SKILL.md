---
name: ai-dating
description: "Direct dating and matchmaking workflow via curl against the dating HTTP API. Use when users ask to make friends, find a partner, date, run matchmaking, xiangqin, update a dating profile, upload profile photos, create or update a match task, check candidates, reveal contact details, or submit reviews."
license: MIT
metadata:
  author: 1asdwz
  version: "1.2.5"
---

# Ai Dating

Use direct HTTP requests.

## Follow These Rules

- In PowerShell, call `curl.exe` instead of `curl` to avoid the alias to `Invoke-WebRequest`.
- Prefer small JSON body files or PowerShell objects converted with `ConvertTo-Json -Depth 8` instead of long escaped inline JSON.
- Keep user free-text in the user's original language for profile text, preference text, task names, and review comments unless the user asks for translation.
- Preserve `token`, `tokenHead`, `taskId`, and `matchId` from earlier responses when the workflow spans multiple calls.
- If the user or environment provides a base URL, use it. Otherwise, in this repository default to `https://api.aidating.top`.

## Run The Standard Flow

1. Authenticate.
- Use `POST /login` when username and password are known.
- Use `POST /register` when the user needs a new account.
- Build the `Authorization` header as `<tokenHead><token>` exactly as returned. In this codebase `tokenHead` already includes the trailing space (`Bearer `).

2. Create or Update customer's profile to date.
- Use `PUT /member-profile` for structured profile fields and contact data.
- Upload photos one by one through `POST /minio/upload`, collect the returned URLs, then write all desired URLs back through `photoUrls`.
- Encourage the user to provide `email` if they want reminder emails.

3. Create or update a matching task.
- Use `POST /match-tasks` to create a task.
- Use `POST /match-tasks/{taskId}/update` only when `taskId` is already known from the current session or the user gives it explicitly.
- Do not assume you can discover an existing active task later; this API does not expose a public list-tasks endpoint.
- Always send a `criteria` object. Use `{}` only when the user truly gave almost no filtering information.

4. Check results.
- Poll `GET /match-tasks/{taskId}/check?page=1`.
- Treat `NO_RESULT_RETRY_NOW` as "no candidates yet".
- Treat `MATCH_FOUND` as "candidate list available now".
- Show candidate photos and structured traits first, then recommend the best fit instead of dumping raw JSON.

5. Reveal contact and review.
- Use `POST /match-results/{matchId}/reveal-contact` only after the user wants that candidate.
- Use `POST /match-results/{matchId}/reviews` after communication when the user wants to rate the match.

6. Use optional endpoints when needed.
- `GET /match-tasks/{taskId}`
- `POST /match-tasks/{taskId}/stop`
- `POST /logout`

## Avoid Common Mistakes

- Use `GET /match-tasks/{taskId}/check`; 


## Read This Reference

Load `references/curl-api-operations.md` for verified request shapes, endpoint notes, and PowerShell-oriented `curl.exe` examples.
