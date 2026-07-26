---
name: bohrium-file
description: "Manage Bohrium personal/share disk files via open.bohrium.com OpenAPI. Use when: user asks to list, inspect, download, upload, create directories, move, copy, delete, decompress, or search files in Bohrium personal disk or project share disk. NOT for: dataset management, compute jobs, dev nodes, images, knowledge bases, or appJob workspace browsing unless an appJob id is explicitly provided."
---

# SKILL: Bohrium File Management

## Overview

Manage Bohrium files through the new `open-platform` gateway:

```text
https://open.bohrium.com/openapi
```

Normal file disks:

| Disk | Path prefix | Required IDs |
|---|---|---|
| Personal disk | `personal/` | `userId`; `projectId=0` usually works for the current user's personal disk |
| Project share disk | `share/` | Real `projectId` + `userId`; requires project permission |

Use **v1 file APIs** for normal personal/share file management. Do not use `/v2/file/iterate` or `/v2/file/download` for personal/share disks.

## Authentication

Read the AccessKey from `BOHR_ACCESS_KEY`. Never print or hardcode it:

```bash
AK="${BOHR_ACCESS_KEY:?missing BOHR_ACCESS_KEY}"
BASE="${BOHR_OPENAPI_BASE:-https://open.bohrium.com/openapi}"
```

Use the same Bearer style as the other Bohrium skills in this repository:

```bash
-H "Authorization: Bearer $AK"
```

The `accessKey: $AK` header is also compatible, but new examples should use `Authorization: Bearer $BOHR_ACCESS_KEY`. Do not send `BOHR_ACCESS_KEY: $AK` as an HTTP header to the public gateway; although open-platform supports that alias in code, the production ingress may filter headers containing underscores. Use `BOHR_ACCESS_KEY` only as the local environment variable name.

Resolve the current identity first:

```bash
curl -sS "$BASE/v1/ak/get" -H "Authorization: Bearer $AK"
```

Use `data.user_id` as `userId` in later calls.

## Route Selection

| Task | Route | Notes |
|---|---|---|
| Current user | `GET /v1/ak/get` | Returns `user_id` and `orgId` |
| List directory | `POST /v1/file/iterate` | Uses `prefix`, not `path` |
| Stat file/dir | `GET /v1/file/stat/{path}` | Query `projectId`, `userId` |
| Metadata | `GET /v1/file/meta/{path}` | Query `projectId`, `userId` |
| Download | `GET /v1/file/download/{path}` | May redirect; use `curl -L` |
| Make directory | `POST /v1/file/mkdir` | Body: `path`, `projectId`, `userId` |
| Move/rename | `POST /v1/file/move` / `/mover` | Body: `sourcePath`, `destinationPath` |
| Copy | `POST /v1/file/copy` / `/copyr` | Body: `sourcePath`, `destinationPath` |
| Delete | `DELETE /v1/file/delete/{path}` / `/deleter/{path}` | Query `projectId`, `userId` |
| Decompress | `POST /v1/file/decompress` | Body: `filePath`, `dirName` |
| Upload credential | `GET /v2/file/upload/binary` | Recommended; returns storage host + upload credential |
| Direct upload | `POST /v1/file/upload/binary` | Legacy one-call flow; returns 307, so curl must use `-L` |

`/v2/file/iterate` and `/v2/file/download` are appJob-workspace APIs only:

```json
{"pathType": "appJob", "pathKey": 12345}
```

Here `pathKey` is the appJob id. The current upstream does not support `pathType: "personal"` or `"share"` and returns `path type not found`.

## List Files

List the personal disk:

```bash
curl -sS -X POST "$BASE/v1/file/iterate" \
  -H "Authorization: Bearer $AK" \
  -H "Content-Type: application/json" \
  -d '{"prefix":"personal/","projectId":0,"userId":27071,"dirSpace":"personal","maxObjects":100,"nextToken":""}'
```

List a project share disk:

```bash
curl -sS -X POST "$BASE/v1/file/iterate" \
  -H "Authorization: Bearer $AK" \
  -H "Content-Type: application/json" \
  -d '{"prefix":"share/","projectId":154,"userId":27071,"dirSpace":"share","maxObjects":100,"nextToken":""}'
```

When `data.hasNext=true`, repeat the same request with `nextToken` set to `data.nextToken`.

## Read Operations

Check whether a path exists:

```bash
curl -sS "$BASE/v1/file/stat/personal/input.txt?projectId=0&userId=27071" \
  -H "Authorization: Bearer $AK"
```

Fetch metadata:

```bash
curl -sS "$BASE/v1/file/meta/personal/input.txt?projectId=0&userId=27071" \
  -H "Authorization: Bearer $AK"
```

Download a file:

```bash
curl -L "$BASE/v1/file/download/personal/input.txt?projectId=0&userId=27071" \
  -H "Authorization: Bearer $AK" \
  -o input.txt
```

Percent-encode URL paths that contain spaces or special characters.

## Write Operations

Create a directory:

```bash
curl -sS -X POST "$BASE/v1/file/mkdir" \
  -H "Authorization: Bearer $AK" \
  -H "Content-Type: application/json" \
  -d '{"path":"personal/new-folder","projectId":0,"userId":27071}'
```

Move or rename:

```bash
curl -sS -X POST "$BASE/v1/file/move" \
  -H "Authorization: Bearer $AK" \
  -H "Content-Type: application/json" \
  -d '{"sourcePath":"personal/old.txt","destinationPath":"personal/new.txt","projectId":0,"userId":27071}'
```

Copy:

```bash
curl -sS -X POST "$BASE/v1/file/copy" \
  -H "Authorization: Bearer $AK" \
  -H "Content-Type: application/json" \
  -d '{"sourcePath":"personal/source.txt","destinationPath":"personal/copy.txt","projectId":0,"userId":27071}'
```

Delete:

```bash
curl -sS -X DELETE "$BASE/v1/file/delete/personal/old.txt?projectId=0&userId=27071" \
  -H "Authorization: Bearer $AK"
```

Use `/mover`, `/copyr`, and `/deleter` for recursive directory operations.

## Upload

Prefer the v2 two-step upload flow. First ask open-platform for a storage gateway credential. The `path` decides whether the file lands in personal or share disk:

```bash
curl -sS "$BASE/v2/file/upload/binary?projectId=0&userId=27071&path=/personal/new.txt" \
  -H "Authorization: Bearer $AK"
```

Example response:

```json
{
  "code": 0,
  "data": {
    "host": "https://tiefblue-nas.dp.tech",
    "Authorization": "Bearer ...",
    "X-Storage-Param": "..."
  }
}
```

This is not a raw OSS upload URL. It is a Bohrium storage/NAS gateway credential. The server resolves `path=/personal/new.txt` or `path=/share/new.txt` to the real storage path and encodes that target inside `X-Storage-Param`; the second upload step writes to the file disk selected by `path`.

Then upload the bytes to the returned `host`:

```bash
curl -sS -X POST "$HOST/api/upload/binary" \
  -H "Authorization: $UPLOAD_AUTHORIZATION" \
  -H "X-Storage-Param: $X_STORAGE_PARAM" \
  --data-binary @local-file.txt
```

Treat `Authorization` and `X-Storage-Param` as secrets. Do not print them in final answers.

The v1 direct upload endpoint also exists:

```bash
curl -L -sS -X POST "$BASE/v1/file/upload/binary?projectId=0&userId=27071&path=personal/new.txt" \
  -H "Authorization: Bearer $AK" \
  --data-binary @local-file.txt
```

Keep `-L`: v1 returns `307 Location: https://tiefblue-nas.dp.tech/api/upload/binary?...`. Without following the redirect, the file is not uploaded.

## Common Pitfalls

- `v1/file/iterate` uses `prefix`, not `path`.
- Move/copy fields are `sourcePath` and `destinationPath`, not `src` / `dst`.
- Do not use `/v2/file/iterate` or `/v2/file/download` for normal personal/share list or download operations.
- Share disk operations require a real `projectId`; `projectId=0` is only suitable for the current user's personal disk.
- The v2 upload endpoint only issues storage/NAS upload credentials; the final destination is determined by the first-step `path`.
- Use `stat` or `meta` before destructive delete, move, or copy operations.
