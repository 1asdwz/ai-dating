# Curl Dating API Operations

Use this reference after `ai-dating` triggers and direct `curl` calls are required.

## Table Of Contents

1. Session setup
2. Endpoint matrix
3. Authentication
4. Profile update and photo upload
5. Match task create and update
6. Task inspection and candidate selection
7. Contact reveal and review
8. Verified backend behaviors

## 1. Session Setup

Prefer `curl.exe` in PowerShell.

```powershell
$BASE_URL = if ($env:AIDATING_BASE_URL) {
  $env:AIDATING_BASE_URL.TrimEnd("/")
} else {
  "https://api.aidating.top"
}
```

Prefer temp JSON files for request bodies:

```powershell
$BodyPath = Join-Path $PWD ".tmp_dating_body.json"
@{ username = "amy_2026" } | ConvertTo-Json -Depth 8 | Set-Content -Encoding utf8 $BodyPath
```

After login or register:

```powershell
$AUTH = "$($Resp.data.tokenHead)$($Resp.data.token)"
$TASK_ID = $null
$MATCH_ID = $null
```

`tokenHead` already includes the trailing space in the current backend configuration, so do not insert an extra one manually.

## 2. Endpoint Matrix

| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/register` | No | Optional body with `username` |
| POST | `/login` | No | Requires `username` and `password` |
| POST | `/logout` | Yes | `GET /logout` also exists, prefer `POST` |
| PUT | `/member-profile` | Yes | Success-only response |
| POST | `/minio/upload` | Yes | One file per request |
| POST | `/match-tasks` | Yes | Requires `taskName` and `criteria` |
| GET | `/match-tasks/{taskId}` | Yes | Returns current task snapshot |
| POST | `/match-tasks/{taskId}/update` | Yes | Same body shape as create; success-only response |
| POST | `/match-tasks/{taskId}/stop` | Yes | Success-only response |
| GET | `/match-tasks/{taskId}/check?page=1` | Yes | Page size is fixed at `10`; max page is `20` |
| POST | `/match-graphql/search` | Yes | Optional advanced query path |
| POST | `/match-results/{matchId}/reveal-contact` | Yes | Returns contact info payload |
| POST | `/match-results/{matchId}/reviews` | Yes | Requires `rating`; success-only response |

## 3. Authentication

### 3.1 Register

Body is optional. Use it only when the user wants a preferred username.

```powershell
@{ username = "amy_2026" } | ConvertTo-Json | Set-Content -Encoding utf8 $BodyPath
$Resp = curl.exe -sS -X POST "$BASE_URL/register" `
  -H "Content-Type: application/json" `
  --data-binary "@$BodyPath" | ConvertFrom-Json
```

Response `data` contains:

- `memberId`
- `username`
- `password`
- `token`
- `tokenHead`

### 3.2 Login

```powershell
@{
  username = "amy_2026"
  password = "123456"
} | ConvertTo-Json | Set-Content -Encoding utf8 $BodyPath

$Resp = curl.exe -sS -X POST "$BASE_URL/login" `
  -H "Content-Type: application/json" `
  --data-binary "@$BodyPath" | ConvertFrom-Json
```

### 3.3 Logout

```powershell
curl.exe -sS -X POST "$BASE_URL/logout" `
  -H "Authorization: $AUTH" | ConvertFrom-Json
```

## 4. Profile Update And Photo Upload

### 4.1 Update Profile

`PUT /member-profile` uses non-null overwrite semantics.

- Sending a field with a value updates it.
- Omitting a field leaves the old value unchanged.
- Sending `photoUrls` replaces the stored photo list with the array you send.
- Sending `educationList` replaces all saved education rows.
- Sending `employmentList` replaces all saved employment rows.

Useful fields:

- `gender`
- `birthday` in `yyyy-MM-dd`
- `heightCm`
- `weightKg`
- `annualIncomeCny`
- `characterText`
- `hobbyText`
- `abilityText`
- `major`
- `nationality`
- `country`
- `province`
- `city`
- `addressDetail`
- `currentLatitude`
- `currentLongitude`
- `currentLocationText`
- `photoUrls`
- `email`
- `phone`
- `telegram`
- `wechat`
- `whatsapp`
- `signal_chat`
- `line`
- `snapchat`
- `instagram`
- `facebook`
- `otherContacts`
- `educationList`
- `employmentList`

Supported aliases from the DTO:

- `hobbiesText` for `hobbyText`
- `mail` for `email`
- `imageUrls` or `profileImageUrls` for `photoUrls`
- `signal` or `signalChat` for `signal_chat`

Example:

```powershell
$ProfileBody = @{
  gender = "male"
  birthday = "1998-08-08"
  heightCm = 180
  annualIncomeCny = 300000
  city = "Hangzhou"
  hobbyText = "badminton, travel, photography"
  characterText = "sincere, steady, humorous"
  abilityText = "cooking, communication, English"
  currentLocationText = "Hangzhou West Lake"
  email = "amy@example.com"
  telegram = "amy_tg"
  wechat = "amy_wechat"
  otherContacts = @{
    xiaohongshu = "amy_xhs"
    discord = "amy#1234"
  }
}

$ProfileBody | ConvertTo-Json -Depth 8 | Set-Content -Encoding utf8 $BodyPath
curl.exe -sS -X PUT "$BASE_URL/member-profile" `
  -H "Authorization: $AUTH" `
  -H "Content-Type: application/json" `
  --data-binary "@$BodyPath" | ConvertFrom-Json
```

### 4.2 Upload Photos

`POST /minio/upload` accepts one file per request. Upload each file separately, collect `data.url`, then send all desired URLs in a later `PUT /member-profile`.

```powershell
$Upload1 = curl.exe -sS -X POST "$BASE_URL/minio/upload" `
  -H "Authorization: $AUTH" `
  -F "file=@C:\photos\me-1.jpg" | ConvertFrom-Json

$Upload2 = curl.exe -sS -X POST "$BASE_URL/minio/upload" `
  -H "Authorization: $AUTH" `
  -F "file=@C:\photos\me-2.jpg" | ConvertFrom-Json

$PhotoUrls = @($Upload1.data.url, $Upload2.data.url)
```

## 5. Match Task Create And Update

### 5.1 Create Task

Request body:

- `taskName`: required
- `criteria`: required object

Prefer the actual DTO field names below.

Structured filter fields:

- `preferredGenderFilter`
- `preferredHeightFilter`
- `preferredIncomeFilter`
- `preferredCityFilter`
- `preferredNationalityFilter`
- `preferredEducationFilter`
- `preferredOccupationFilter`

Text and semantic fields:

- `preferredEducationStage`
- `preferredOccupationKeyword`
- `preferredHobbyText`
- `preferredCharacterText`
- `preferredAbilityText`
- `intention`
- `hobbyEmbeddingMinScore`
- `characterEmbeddingMinScore`
- `abilityEmbeddingMinScore`
- `intentionEmbeddingMinScore`
- `preferredContactChannel`

Use nested objects for filter expressions when possible. The backend setter accepts either an object or a string and normalizes it internally.

Example:

```powershell
$TaskBody = @{
  taskName = "Find long-term partner in Hangzhou"
  criteria = @{
    preferredGenderFilter = @{ eq = "female" }
    preferredHeightFilter = @{ gte = 165; lte = 178 }
    preferredIncomeFilter = @{ gte = 200000 }
    preferredCityFilter = @{ in = @("Hangzhou", "Shanghai") }
    preferredEducationStage = "Bachelor or above"
    preferredOccupationKeyword = "Product"
    preferredHobbyText = "reading, travel"
    preferredCharacterText = "kind, positive"
    preferredAbilityText = "strong communication"
    intention = "long-term relationship"
  }
}

$TaskBody | ConvertTo-Json -Depth 8 | Set-Content -Encoding utf8 $BodyPath
$TaskResp = curl.exe -sS -X POST "$BASE_URL/match-tasks" `
  -H "Authorization: $AUTH" `
  -H "Content-Type: application/json" `
  --data-binary "@$BodyPath" | ConvertFrom-Json

$TASK_ID = $TaskResp.data.taskId
```

Notes:

- `criteria` cannot be omitted.
- `preferredEducationStage` and `preferredOccupationKeyword` are convenience fields; the service converts them into `contains` filters internally.
- If `preferredHobbyText`, `preferredCharacterText`, `preferredAbilityText`, or `intention` are present and no min score is supplied, create-task logic defaults that min score to `0.1`.

### 5.2 Update Task

Use the same body shape as create:

```powershell
$UpdateBody = @{
  taskName = "Update criteria - Hangzhou or Shanghai"
  criteria = @{
    preferredGenderFilter = @{ eq = "female" }
    preferredHeightFilter = @{ gte = 163; lte = 180 }
    preferredCityFilter = @{ in = @("Hangzhou", "Shanghai") }
    preferredCharacterText = "independent, optimistic"
    intention = "serious long-term relationship"
  }
}

$UpdateBody | ConvertTo-Json -Depth 8 | Set-Content -Encoding utf8 $BodyPath
curl.exe -sS -X POST "$BASE_URL/match-tasks/$TASK_ID/update" `
  -H "Authorization: $AUTH" `
  -H "Content-Type: application/json" `
  --data-binary "@$BodyPath" | ConvertFrom-Json
```

There is no public list endpoint to recover forgotten task IDs, so keep `taskId` when you create a task.

## 6. Task Inspection And Candidate Selection

### 6.1 Get Task

```powershell
curl.exe -sS "$BASE_URL/match-tasks/$TASK_ID" `
  -H "Authorization: $AUTH" | ConvertFrom-Json
```

### 6.2 Check Candidates

```powershell
$CheckResp = curl.exe -sS "$BASE_URL/match-tasks/$TASK_ID/check?page=1" `
  -H "Authorization: $AUTH" | ConvertFrom-Json
```

Read:

- top-level `watchStatus`
- `data.watchStatus`
- `data.page`
- `data.total`
- `data.totalPages`
- `data.candidates`

`check` returns a reduced `MatchCandidateView`. Useful fields include:

- `matchId`
- `memberId`
- `username`
- `nickname`
- `gender`
- `heightCm`
- `annualIncomeCny`
- `city`
- `hobbyText`
- `character`
- `abilityText`
- `photoUrls`
- `currentLocationText`
- `rankScore`
- `exactMatchCount`
- `exactConditionCount`
- `confirmedViolationCount`
- `dailyExposureCount`
- `profileUpdatedAt`

Selection guidance:

- Show photos first when available.
- Prefer higher `rankScore`, fewer confirmed violations, and a better exact-match ratio.
- Explain tradeoffs instead of presenting only one opaque winner.
- Do not claim contact information is available until `reveal-contact` succeeds.

### 6.3 Optional GraphQL Search

Use `POST /match-graphql/search` only when the user needs a constrained candidate query or richer candidate fields than `/check` returns.

Requirements enforced by service code:

- `query` must contain `matchCandidates`
- `variables.taskId` is required
- `variables.limit` defaults to `20`
- `variables.criteria` is optional and accepts the same structure as task criteria

Example:

```powershell
$GraphqlBody = @{
  query = "query MatchCandidates(`$taskId:Long!, `$limit:Int){ matchCandidates(taskId:`$taskId, limit:`$limit){ matchId memberId city rankScore compatibilityScore } }"
  variables = @{
    taskId = $TASK_ID
    limit = 20
  }
}

$GraphqlBody | ConvertTo-Json -Depth 8 | Set-Content -Encoding utf8 $BodyPath
curl.exe -sS -X POST "$BASE_URL/match-graphql/search" `
  -H "Authorization: $AUTH" `
  -H "Content-Type: application/json" `
  --data-binary "@$BodyPath" | ConvertFrom-Json
```

## 7. Contact Reveal And Review

### 7.1 Reveal Contact

```powershell
$RevealResp = curl.exe -sS -X POST "$BASE_URL/match-results/$MATCH_ID/reveal-contact" `
  -H "Authorization: $AUTH" | ConvertFrom-Json
```

Response `data` may contain:

- `matchId`
- `targetMemberId`
- `phone`
- `telegram`
- `wechat`
- `whatsapp`
- `signal_chat`
- `line`
- `snapchat`
- `instagram`
- `facebook`
- `otherJson`

### 7.2 Submit Review

```powershell
$ReviewBody = @{
  rating = 5
  comment = "Communication was smooth and values were aligned"
}

$ReviewBody | ConvertTo-Json -Depth 8 | Set-Content -Encoding utf8 $BodyPath
curl.exe -sS -X POST "$BASE_URL/match-results/$MATCH_ID/reviews" `
  -H "Authorization: $AUTH" `
  -H "Content-Type: application/json" `
  --data-binary "@$BodyPath" | ConvertFrom-Json
```

Rules from the DTO and service:

- `rating` is required and must be `1..5`
- the same user can review the same `matchId` only once
- ratings `<= 2` automatically create a pending violation case

## 8. Verified Backend Behaviors

These points are confirmed against controller and service code in this repository.

- `GET /match-tasks/{taskId}/check` is the active polling endpoint; the API docs mention `/watch` in one place, but no public `/watch` controller exists.
- `PUT /member-profile` returns `CommonResult.success()` with `data = null`.
- `POST /match-tasks/{taskId}/update` returns `CommonResult.success()` with `data = null`.
- `POST /match-tasks/{taskId}/stop` returns `CommonResult.success()` with `data = null`.
- `POST /match-results/{matchId}/reviews` returns `CommonResult.success()` with `data = null`.
- `POST /minio/upload` returns `CommonResult` whose `data.url` is the uploaded image URL.
- `check` rejects inactive tasks and validates `page >= 1` and `page <= 20`.
- `check` page size is fixed to `10`, with at most `200` candidates available through pagination.
- The backend auto-generates embeddings from text profile and criteria fields when embedding is configured. If embedding is misconfigured, calls using semantic text fields can fail even when the rest of the payload is valid.
