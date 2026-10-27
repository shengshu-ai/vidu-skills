# Reference Elements and Creations

Use with `vidu-cli v0.2.36+`. An element is a reusable reference; a creation is an output artifact of a task. IDs in examples must be taken from responses and used with an account that can access them.

## ID Sources

| ID | Obtain from | Use for |
|---|---|---|
| `task_id` | Task submission/history | Query, wait, cancel, delete history, rerun |
| Element `id` and `version` | `element create/get/list/search/favorites` | `--material "NAME:ID:VERSION"` on a supported generation model |
| Numeric `creation_id` | `task get TASK_ID --details` or `creation list` | Extend, enhance/upscale, favorite a creation |
| Personal `voice_id` (`voice_...`) | `voice create/list` | Personal voice status/update and TTS; see [voices](voices.md) |

Never substitute a task ID for a creation ID. Use the selected element's current response version instead of assuming version 1. `--material` uses an element ID/version, not the community share ID.

## Create an Element

For a one-off reference image, pass `--image` directly to generation. Use `element create` when the user wants a saved, reusable subject.

```bash
vidu-cli element check --name "my_character"
vidu-cli element preprocess --name "my_character" --image ./portrait.jpg --type user
vidu-cli element create --name "my_character" --image ./portrait.jpg \
  --image ./profile.jpg --description "A young explorer" --style "realistic"
```

`create` already performs name check → image upload → preprocess → create. Separate `check` and `preprocess` are available when the user needs those results independently; do not run them redundantly for every create.

- Create/preprocess require 1–3 images; repeat `--image`. Inputs may be local files, image URLs, or `ssupload:?id=...`.
- Creation accepts `--name`, optional `--description` (1–1280 characters), and optional `--style` (≤64 characters); omitted description/style use preprocessing results. `--modality` defaults to `image`, and `--type` defaults to `user`.
- Name availability and account permissions are validated by the service. Returns element `id` and `version`, not a creator `task_id`; no duration or task wait option applies.

For generation, use one material argument per element, quoting it as a unit:

```bash
# Set these variables to element response values; the name matches the prompt marker.
vidu-cli task submit --type character2video --prompt "[@Explorer] walks through a forest" \
  --material "Explorer:$ELEMENT_ID:$ELEMENT_VERSION" --model-version 3.2_a \
  --duration 5 --resolution 1080p --wait --output ./results
```

Drama and Wan 3 do not accept material references. See the [model matrix](parameters.md) before reusing an element in a different model.

## Personal Elements

```bash
vidu-cli element list --keyword "explorer" --pagesz 20
vidu-cli element list --keyword "explorer" --page-token "$NEXT_PAGE_TOKEN" --pagesz 20
vidu-cli element list --page 0 --pagesz 20
vidu-cli element get "$ELEMENT_ID"
vidu-cli element update "$ELEMENT_ID" --name "New explorer" --label 1 --label 2 \
  --recaption "An explorer in a blue coat"
vidu-cli element update "$ELEMENT_ID" --clear-labels
vidu-cli element delete "$ELEMENT_ID"
```

- `list` returns `elements` and `next_page_token`; use the returned continuation token. `--pagesz` is 10–100, default 20. Omit both page selectors for the first page.
- `--page` is zero-based and conflicts with `--page-token`. Numeric page compatibility walks cursors internally, so deep pages can require several requests. Do not invent a numeric offset API for personal elements.
- `get` fetches the latest visible element. Updating requires at least one of name, labels, clear-labels, or recaption.
- `--label` takes a numeric label ID and is repeatable; supplied IDs **replace the whole label set**, not append to it. IDs 1/2 above are examples, not a label catalog. Use labels available to the account. `--label` conflicts with `--clear-labels`.
- `--recaption` changes the description while retaining style. `--description` and `--style` are create options, not update options.
- Shared submissions, version restrictions, ownership, and deletion permissions are enforced by the server; surface failures without inventing alternate write paths.

## Community Search and Favorites

```bash
vidu-cli element search --keyword "summer" --sort-by recommend --pagesz 20
vidu-cli element search --keyword "summer" --page-token "$NEXT_PAGE_TOKEN" --pagesz 20
vidu-cli element favorite "$ELEMENT_ID"
vidu-cli element favorites --pagesz 20
vidu-cli element favorites --page-token "$NEXT_PAGE_TOKEN" --pagesz 20
vidu-cli element unfavorite "$ELEMENT_ID"
```

Search requires `--keyword`, also accepts `--category`, `--topic`, `--sort-by` (default `recommend`), and uses `--page-token`. Use real service category/topic values when those filters are requested. `search` / `favorites` accept `--pagesz` 1–100, default 20.

`element favorites` is a separate cursor list and accepts only pagination, not keyword/category/topic filters. `favorite` and `unfavorite` take the element ID; the CLI resolves its share ID internally. Present name, ID/version, description, and category when returned; do not fabricate missing fields.

## Creations

```bash
vidu-cli creation list --page 0 --pagesz 20
vidu-cli creation list --favorites-only --page 0 --pagesz 20
vidu-cli creation favorite "$CREATION_ID"
vidu-cli creation unfavorite "$CREATION_ID"
```

Creation lists use zero-based numeric `--page` (default 0) and `--pagesz` 1–100 (default 20), returning `creations` and `total`; there is no page-token flag. Favorites may include other users' accessible creations. Use the positive numeric creation ID from these results or `task get --details`; the CLI normalizes returned IDs as `creation_id`.

For extension and image/video upscaling from a selected creation, read [task workflows](task_workflows.md). For the numeric-page credit ledger, see `quota records` in [parameters](parameters.md).
