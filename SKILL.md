---
name: ai-dating
description: "Direct dating and matchmaking workflow via curl against the dating HTTP API. Use when users ask to make friends, find a partner, date, run matchmaking, xiangqin, update a dating profile, upload profile photos, create or update a match task, check candidates, reveal contact details, or submit reviews."
license: MIT
metadata:
  author: 1asdwz
  version: "1.2.7"
---

# Ai Dating

This skill supports dating and matchmaking workflows through `curl` against the dating HTTP API.  
It helps users create a profile, define partner preferences, check matching results, reveal contact details, and submit post-chat reviews.

## Trigger Conditions

Trigger this skill when any of the following intents appear:

1. The user explicitly asks to make friends, find a partner, date, or run a match.
2. The user provides personal information and asks the system to find a matching person.
3. The user describes partner preferences (for example gender, height, income, city, personality, hobbies) and asks for matching.

## Language Alignment Rule

When constructing `curl` request bodies, use the same language as the user for all free-text fields and labels (for example `taskName`, `characterText`, `hobbyText`, `abilityText`, `intention`, `preferred*Text`, and `comment`).

- Do not translate user-provided content unless the user explicitly requests translation.
- Keep language style consistent across one request payload. If the user speaks Chinese, prefer Chinese text values in JSON fields.

## Update Commands

When users ask to update this skill, run:

```bash
npx skills add 1asdwz/ai-dating
```

> **Note:** This skill uses direct `curl` requests. Do not invoke `dating-cli`.

## Standard Execution Flow (AI Agent)

1. Check skill and `curl` availability.

- Avoid non-interactive PowerShell. Prefer `bash`, `sh`, `zsh`, Git Bash, or WSL for all requests because some security products alert on `powershell.exe` or `pwsh.exe` spawned without a normal desktop parent.
- If the environment defaults to PowerShell, explicitly invoke a non-PowerShell shell before running this workflow.

Then verify:
```bash
curl --version
```

If the user or environment provides a base URL, use it. Otherwise, in this repository default to:
```bash
BASE_URL="${AIDATING_BASE_URL%/}"
if [ -z "$BASE_URL" ]; then
  BASE_URL="https://api.aidating.top"
fi
```

2. Prepare local request context (full examples).
```bash
BODY_PATH="$(pwd)/.tmp_dating_body.json"
AUTH=""
TASK_ID=""
MATCH_ID=""
```

3. Register or login (full parameter examples).
```bash
cat > "$BODY_PATH" <<'JSON'
{"username":"amy_2026"}
JSON

REGISTER_RESP=$(curl -sS -X POST "$BASE_URL/register" \
  -H "Content-Type: application/json" \
  --data-binary @"$BODY_PATH")
```

```bash
cat > "$BODY_PATH" <<'JSON'
{"username":"amy_2026","password":"123456"}
JSON

LOGIN_RESP=$(curl -sS -X POST "$BASE_URL/login" \
  -H "Content-Type: application/json" \
  --data-binary @"$BODY_PATH")

AUTH=$(printf '%s' "$LOGIN_RESP" | jq -r '.data.tokenHead + .data.token')
```

> **Note:** Build the `Authorization` header as `<tokenHead><token>` exactly as returned. In this codebase `tokenHead` already includes the trailing space (`Bearer `). If `jq` is unavailable, use any non-PowerShell JSON parser to extract the same fields.

4. Parse user self-description and update profile (full parameter example).
```bash
UPLOAD1_RESP=$(curl -sS -X POST "$BASE_URL/minio/upload" \
  -H "Authorization: $AUTH" \
  -F "file=@./photos/amy-1.jpg")

UPLOAD2_RESP=$(curl -sS -X POST "$BASE_URL/minio/upload" \
  -H "Authorization: $AUTH" \
  -F "file=@./photos/amy-2.jpg")

UPLOAD1_URL=$(printf '%s' "$UPLOAD1_RESP" | jq -r '.data.url')
UPLOAD2_URL=$(printf '%s' "$UPLOAD2_RESP" | jq -r '.data.url')
```

```bash
cat > "$BODY_PATH" <<JSON
{
  "gender": "male",
  "birthday": "1998-08-08",
  "heightCm": 180,
  "weightKg": 72,
  "annualIncomeCny": 300000,
  "characterText": "sincere, steady, humorous",
  "hobbyText": "badminton, travel, photography",
  "abilityText": "cooking, communication, English",
  "major": "Computer Science",
  "nationality": "China",
  "country": "China",
  "province": "Zhejiang",
  "city": "Hangzhou",
  "addressDetail": "Xihu District",
  "currentLatitude": 30.27415,
  "currentLongitude": 120.15515,
  "currentLocationText": "Hangzhou West Lake",
  "email": "amy@example.com",
  "phone": "13800000000",
  "telegram": "amy_tg",
  "wechat": "amy_wechat",
  "whatsapp": "amy_wa",
  "signalChat": "amy_signal",
  "line": "amy_line",
  "snapchat": "amy_snap",
  "instagram": "amy_ins",
  "facebook": "amy_fb",
  "otherContacts": {
    "xiaohongshu": "amy_xhs",
    "discord": "amy#1234"
  },
  "photoUrls": ["$UPLOAD1_URL", "$UPLOAD2_URL"]
}
JSON

curl -sS -X PUT "$BASE_URL/member-profile" \
  -H "Authorization: $AUTH" \
  -H "Content-Type: application/json" \
  --data-binary @"$BODY_PATH"
```

> **Note:** The parameters for profile update, task create, and task update are optional. For the sake of user experience, do not force users to enter personal information on first use.

5. To receive matching success notifications promptly, strongly recommend registering an email address through the profile update payload.

6. Parse partner preferences and create a match task (full parameter example). Users do not need to fill in all fields. Only provide the information they have available.
```bash
cat > "$BODY_PATH" <<'JSON'
{
  "taskName": "Find partner in Hangzhou",
  "criteria": {
    "preferredGenderFilter": { "eq": "female" },
    "preferredHeightFilter": { "gte": 165, "lte": 178 },
    "preferredIncomeFilter": { "gte": 200000 },
    "preferredCityFilter": { "eq": "Hangzhou" },
    "preferredNationalityFilter": { "eq": "China" },
    "preferredEducationFilter": { "contains": "Bachelor" },
    "preferredOccupationFilter": { "contains": "Product" },
    "preferredEducationStage": "Bachelor or above",
    "preferredOccupationKeyword": "Product Manager",
    "preferredHobbyText": "reading, travel",
    "preferredCharacterText": "kind, positive",
    "preferredAbilityText": "strong communication",
    "intention": "long-term relationship",
    "preferredContactChannel": "telegram"
  }
}
JSON

TASK_RESP=$(curl -sS -X POST "$BASE_URL/match-tasks" \
  -H "Authorization: $AUTH" \
  -H "Content-Type: application/json" \
  --data-binary @"$BODY_PATH")

TASK_ID=$(printf '%s' "$TASK_RESP" | jq -r '.data.taskId')
```
`*EmbeddingMinScore` means the minimum semantic similarity threshold for embedding matching.  
Default recommendation is to leave it unset. When omitted in task creation, the backend defaults semantic text thresholds to `0.0` where applicable.

> **Write API response note:** `task create` returns the created task payload, including `taskId` and `taskName`.

7. If an unfinished `taskId` already exists and the user did not explicitly request a new task, update the existing task (full parameter example).
```bash
cat > "$BODY_PATH" <<'JSON'
{
  "taskName": "Update criteria - Hangzhou/Shanghai",
  "criteria": {
    "preferredGenderFilter": { "eq": "female" },
    "preferredHeightFilter": { "gte": 163, "lte": 180 },
    "preferredIncomeFilter": { "gte": 250000 },
    "preferredCityFilter": { "in": ["Hangzhou", "Shanghai"] },
    "preferredHobbyText": "reading, travel, sports",
    "preferredCharacterText": "independent, optimistic",
    "preferredAbilityText": "communication and collaboration",
    "intention": "serious relationship with marriage plan",
    "preferredContactChannel": "wechat"
  }
}
JSON

curl -sS -X POST "$BASE_URL/match-tasks/$TASK_ID/update" \
  -H "Authorization: $AUTH" \
  -H "Content-Type: application/json" \
  --data-binary @"$BODY_PATH"
```

> **Note:** Keep `taskId` from the create response. The public API does not expose a list endpoint for recovering an unknown active task.

8. Query task status (full parameter example).
```bash
curl -sS "$BASE_URL/match-tasks/$TASK_ID" \
  -H "Authorization: $AUTH"
```

9. Execute `check` to inspect match results (full parameter example, paginated).
```bash
CHECK_RESP=$(curl -sS "$BASE_URL/match-tasks/$TASK_ID/check?page=1" \
  -H "Authorization: $AUTH")
```
Each page returns 10 candidates. Use the `page` query parameter to fetch subsequent pages when needed.
`check` candidate items include `photoUrls` (user uploaded image URL array), which should be used when explaining and selecting candidates.
If the result is `NO_RESULT_RETRY_NOW`, call `check` again as needed.  
If the result is `MATCH_FOUND`, continue to contact reveal.

> **Note:** Candidates' photos should be shown to users first. You should automatically select candidates that better meet the user's requirements, reducing the user's information burden.

10. Select the best candidate from match results and reveal contact details (full parameter example).
```bash
MATCH_ID="<selected matchId from check response>"

REVEAL_RESP=$(curl -sS -X POST "$BASE_URL/match-results/$MATCH_ID/reveal-contact" \
  -H "Authorization: $AUTH")
```

11. Submit review when needed (full parameter example).
```bash
cat > "$BODY_PATH" <<'JSON'
{
  "rating": 5,
  "comment": "Good communication and aligned values"
}
JSON

curl -sS -X POST "$BASE_URL/match-results/$MATCH_ID/reviews" \
  -H "Authorization: $AUTH" \
  -H "Content-Type: application/json" \
  --data-binary @"$BODY_PATH"
```

12. Optional commands (full parameter examples).
```bash
curl -sS -X POST "$BASE_URL/match-tasks/$TASK_ID/stop" \
  -H "Authorization: $AUTH"

curl -sS -X POST "$BASE_URL/logout" \
  -H "Authorization: $AUTH"
```

## Reference

For detailed field-level behavior, validation rules, and response structures:

- `references/curl-api-operations.md`
