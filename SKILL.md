---
name: decap-to-pages-cms-config
description: Converts Decap CMS (formerly Netlify CMS) config.yml to Pages CMS .pages.yml format. Use when migrating from Decap/Netlify CMS to Pages CMS, when the user asks to convert config, or when editing config for Pages CMS based on an existing Decap config.
---

# Decap CMS to Pages CMS Config Conversion

Convert [Decap CMS](https://decapcms.org/docs/intro/) `config.yml` to [Pages CMS](https://pagescms.org/docs/) `.pages.yml`. Output file is **`.pages.yml`** at the repo root (or the branch/content root Pages CMS uses).

## Quick conversion steps

1. **Reuse with YAML anchors**  
   Pages CMS supports YAML reusables. Keep Decap-style anchors: define once with `&NAME` (e.g. `meta: &META ...`) and reuse with `*NAME` in `fields` (e.g. `- *META`). Do not use a separate `components:` section—Pages docs for that are out of date.

2. **Map top-level structure**
   - `media_folder` + `public_folder` → `media` (see [reference.md](reference.md)).
   - `collections` → `content` array. Ignore `backend`, `local_backend`, `publish_mode`, `slug`, etc. (Pages handles these differently or not in config).
   - **Always add** `settings.content.merge: true` so saves merge with existing file content instead of overwriting (preserves frontmatter/fields not in the schema).

3. **Map each collection**
   - **Folder collection** (`folder: path`): `type: collection`, `path:` = same folder path.
   - **File collection** (`files: [...]`): each file → separate `type: file` content entry with `path:` to that file.
   - `filter` (e.g. `field: layout`, `value: x`): same folder can appear as multiple content entries with different `path` and/or `view.default.search`; document that filtering is by frontmatter and may need manual handling in Pages.
   - `identifier_field` → use as `view.primary` and in `filename` if needed.
   - `extension` → influences `format` (e.g. `md` → `yaml-frontmatter`).
   - `slug` template → `filename` (replace `{{slug}}` with `{primary}`, `{{year}}` → `{year}`, etc.).
   - `meta.path` (Decap path template) → Pages `filename` pattern (e.g. `'{year}-{month}-{day}-{primary}.md'` or custom).
   - `nested` / path template → `view.node` and `view.layout: tree` if you want tree view; `filename` for nested paths.

4. **Map fields**
   - Use the **widget → type** table in [reference.md](reference.md).
   - Decap `widget: X` → Pages `type: X` (different names: string, text, number, boolean, image, file, date, select, object, etc.).
   - **Required attributes:** Decap defaults fields to required unless `required: false`. When converting, explicitly set `required:` so validation is clear: for identifier/primary fields (e.g. `title` when used as `view.primary`), slug-derived fields, and main body set `required: true`; for optional meta, navigation, images, captions, etc. set `required: false`. Pages uses `required: true` to block saving when empty; omit or `required: false` for optional fields.
   - Decap `default` → Pages `default`.
   - Decap `pattern: [regex, message]` → Pages `pattern: { regex: '...', message: '...' }`.
   - Decap `hint` → Pages `description`. Make sure these have quote marks to prevent any errors when using colons.
   - **Date with time:** Decap `widget: datetime` (or content like `2025-05-23T17:34:57.853Z`) → Pages `type: date` with `options: { time: true }`. Omit `options.time` for date-only. If content uses full ISO with seconds/milliseconds/UTC (e.g. `2025-05-23T17:34:57.853Z`), add `format: "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"` (date-fns format); default with time is only `yyyy-MM-dd'T'HH:mm` and won’t match. You may need to check the markdown file to make sure the format matches. The format needs to be within the options key.

5. **Blocks (list with types)**
   - Decap: `widget: list` with `types: [ { label, name, widget: object, fields: [...] }, ... ]` (block-style list).
   - Pages: `type: block`, `list: true`, `blocks: [ { name:, label:, fields: [...] }, ... ]`. Inline `fields` per block; reuse shared objects (e.g. button) via YAML alias (`*BUTTON`). Omit `blockKey`—Normally 'type'. Include this as an attribute.
   - **Blocks are a list:** most list types can be collapsible; add `list: { collapsible: { collapsed: true, summary: '{fields.type}' } }` (or similar) so each block entry is collapsible with a clear summary.

6. **Lists (repeatable, single type)**
   - Decap: `widget: list` with `fields:` or `field:` (no `types`).
   - Pages: same field definition with `list: true`. Use `list: { min, max, default, collapsible }` for min/max and collapse/summary (map Decap `summary`, `collapsed`, `min`, `max`).
   - **Most list types can be collapsible** in Pages. Add `list: { collapsible: { collapsed: true, summary: '...' } }` so list entries render collapsed by default; use a summary template (e.g. `{fields.title}`, `{fields.type}`, `{index}`) so each entry is identifiable when collapsed.

7. **Objects**
   - Decap: `widget: object`, `fields: [...]`, optional `collapsed`, `summary`.
   - Pages: `type: object`, `fields: [...]`. Use `list.collapsible: { collapsed:, summary: }` for list-of-objects.

8. **Body / markdown**
   - Decap: field `name: body`, `widget: markdown` → Pages: `name: body`, `type: rich-text` (or `text` if you want plain markdown). Pages uses `body` as reserved for main content in frontmatter formats.

9. **View (collection list)**
   - Pages `view:`: `primary`, `fields`, `sort`, `search`, `default.sort`, `default.order`, `default.search`, `layout: list|tree`, `node` for tree/nested.

10. **Media**
   - Single: `media: inputPath` or `media: { input:, output: }` (Decap `media_folder` → `input`, `public_folder` → `output`).
   - Multiple: `media: [ { name:, label:, input:, output:, path:, extensions:, categories: } ]`.

## Output layout

Use **YAML anchors and aliases** for reusable field groups (e.g. `&META` / `*META`). Do not use a `components:` section.

```yaml
# .pages.yml
media: { input: ..., output: ... }

settings:
  content:
    merge: true

meta: &META
  name: meta
  label: Meta
  type: object
  fields: [...]

button: &BUTTON
  name: button
  label: Button
  type: object
  fields: [...]

content:
  - name: pages
    label: Pages
    type: collection
    path: src
    format: yaml-frontmatter
    filename: '{primary}/index.md'
    view: { primary: title, fields: [title], ... }
    fields:
      - ...
      - *META
  - name: blog
    label: Blog
    type: collection
    path: src/blog/items
    fields: [...]
```

## Checklist before finishing

- [ ] All collections converted to `content` entries (`collection` or `file`).
- [ ] All fields use Pages `type` and key names (`label`, `name`, `required`, `default`, `pattern`, `description`).
- [ ] Block lists use `type: block`, `list: true`, `blocks:`; reuse shared fields with YAML aliases (e.g. `*BUTTON`).
- [ ] Media matches your repo layout (`input` = where files live, `output` = URL path).
- [ ] `filename` and `view.primary` match your identifier field and desired URLs.
- [ ] Reusable field groups use YAML anchors/aliases (e.g. `&META` / `*META`), not a `components:` section.
- [ ] `settings.content.merge: true` is included so saves merge with existing files.
- [ ] Required attributes set: `required: true` on primary/identifier and main content fields (e.g. title, body); `required: false` on optional meta, nav, images, captions.

## References

- Full **widget → type** and **option** mappings: [reference.md](reference.md).
- Decap: [Configuration options](https://decapcms.org/docs/configuration-options/), [Widgets](https://decapcms.org/docs/widgets/), [Folder collections](https://decapcms.org/docs/collection-folder/).
- Pages: [Configuration](https://pagescms.org/docs/configuration), [Block field](https://pagescms.org/docs/configuration/block-field), [Fields](https://pagescms.org/docs/configuration#fields).
