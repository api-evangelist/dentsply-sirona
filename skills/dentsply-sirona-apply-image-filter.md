---
name: Apply a Dentsply Sirona intraoral image filter
description: >-
  Take a 16-bit grayscale intraoral image — uploaded, or referenced from a live modality session —
  and apply the Select, Supreme or AE filter using the Intraoral Imaging Filters API, then clean up
  the transient resource.
api: openapi/dentsply-sirona-intraoral-filters-openapi.yml
generated: '2026-09-06'
method: generated
source: openapi/dentsply-sirona-intraoral-filters-openapi.yml
operations:
  - createImage
  - modalityImage
  - getImage
  - getImageMedia
  - unmapImage
  - filterSelect
  - filterSupreme
  - filterAE
  - deleteImage
---

# Apply an intraoral image filter

## Before you start

- Local service: `https://localhost:43809/api/dsio/filters/v1`, same workstation as the Modality
  service. The Filters contract declares **no security scheme** — it is a loopback service.
- Image resources are **transient**. Each carries an `expires` timestamp after which it is deleted
  automatically. The provider does not publish the retention duration, so do not depend on one.
- No idempotency: a retried `createImage` creates a **second** resource you must delete separately.

## Steps

1. **Create the image resource**, one of two ways:
   - `createImage` — multipart upload of a **16-bit grayscale PNG** plus an `ImageInfo` part.
     A `415` means the file is not a 16-bit PNG; a `400` means bad input parameters.
   - `modalityImage` — reference an image already captured in a live Modality acquisition session.
     Cheaper, but the resource "will only be available as long as the modality session is active".
     A `404` means the modality session or image was not found.
2. **Inspect it if you need to.** `getImage` returns the `ImageResource` — `id`, `media` URL,
   `createdOn`, `expires`, `imageInfo`, and `modalitySession` when it came from step 1b.
   `getImageMedia` returns the media itself.
3. **Apply a filter.** All three return `image/png`:
   - `filterSelect` with `SelectFilter` parameters
   - `filterSupreme` with `SupremeFilter` parameters
   - `filterAE` with `AEFilter` parameters
   Handle `400` (invalid filter parameters), `404` (resource gone), and `410 Gone` — the resource
   record still exists but its media has expired or been reaped, so re-create it from step 1.
4. **Undo a filter for display.** `unmapImage` returns the unmapped raw rendering. That is the
   practical undo; the filter POSTs compute and return, they do not mutate stored state.
5. **Delete the resource.** `deleteImage` returns `204`. The provider's own guidance: "When
   finished with filter operations, this resource should be deleted... Otherwise, the resource will
   be automatically deleted after the `expires` timestamp." Do not rely on expiry.

## Note on schema drift

`ImageInfo` and `BinningMode` are defined **independently** in the Filters and Modality contracts
rather than shared. Do not assume the two definitions are identical; validate against the contract
you are actually calling. See `data-model/dentsply-sirona-data-model.yml`.
