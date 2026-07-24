# webkit demo - guestbook

A small but complete app built on the [webkit](../) framework: a guestbook you
can post to.

```bash
cd demo
ecko app.ecko          # then open http://localhost:8090
```

## What it shows

| Feature | Where |
| --- | --- |
| `webkit.app` assembly (routes + static + middleware + errors) | bottom of `app.ecko` |
| Native templates for views | `layout` / `message_item` / `flash_note` |
| **Escaping every piece of user data** with `webkit.e` | `home_page` |
| Validated `POST` form → flash → redirect | `create_message` |
| In-memory state shared across workers via a `cell` | `messages` |
| Route group (blueprint) for a JSON API | `/api/messages` |
| Static files with `cache-control` | `webkit.static` + `webkit.cache_control` |
| Security headers + a custom 404 | `app({ security: true, errors: … })` |
| Health check | `/health` |

## Notes

- This file lives inside the webkit repo, so it imports the package source as
  `main` (`import "../main.ecko"`). In a real app you'd `import webkit` and call
  `webkit.app(...)`, granting the package `["fs:read", "net"]` in `ecko.json`.
- Try it: post a message with `<script>` in your name - it renders escaped,
  because every user value goes through `webkit.e`. The `/api/messages` JSON
  keeps the raw value (JSON handles its own encoding; escaping is an HTML
  concern).
- State is in memory (a `cell`), so it resets when the server restarts.
