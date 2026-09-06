---
name: Retrieve intraoral X-ray exposure dose records
description: >-
  Read exposure dose data from an intraoral X-ray tube generator so it can be stored with the
  patient's media and dental record, using the Intraoral Exposure API.
api: openapi/dentsply-sirona-intraoral-exposure-openapi.yml
generated: '2026-09-06'
method: generated
source: >-
  openapi/dentsply-sirona-intraoral-exposure-openapi.yml,
  https://github.com/dsimaging/io-exposure-service/blob/main/docs/Recommendations.md
operations:
  - getAllGenerators
  - getGenerator
  - getExposures
  - getExposure
---

# Retrieve exposure dose records

## What this API actually is

A **specification Dentsply Sirona published for X-ray generator vendors to implement**, not a
service Dentsply Sirona hosts. There is no first-party base URL: the only `servers[]` entry in the
contract is a SwaggerHub auto-mock under an individual engineer's account. The repository is
**archived**. Treat this as a protocol you may find implemented on a local network device.

The whole API is **read-only** — four GETs, no writes. Idempotency, dry-run and reversibility are
all `na` here.

## Finding an endpoint

There is no published host. The provider recommends:

- **URL-path versioning**: "select an appropriate endpoint for the service that includes the API
  version", e.g. `https://xraygen:443/api/ioexp/v1/`.
- **mDNS discovery**: announce and query the service name `_io-exposure._tcp`, and read the
  endpoint per version from a TXT record shaped `v1.0=/api/v1/;v1.1=/api/v1.1/;v2=/api/v2`.

Auth is an `X-API-Key` header, applied globally by the contract. The key is issued by whoever
operates the generator service.

## Steps

1. **List generators.** `getAllGenerators` returns every intraoral X-ray tube generator the service
   manages.
2. **Identify one.** `getGenerator` with the generator `id`. `404` means it does not exist.
3. **List its exposures.** `getExposures` — the only paginated operation in Dentsply Sirona's
   entire published surface. Query with `skip`, `limit` (**maximum 50**), `startTime` and
   `endTime` (RFC 3339). The response envelope carries `total`, `skip`, `limit` and `exposures[]`.
   Page by advancing `skip` until `skip + len(exposures) >= total`. A `400` means invalid search
   parameters.
4. **Read one exposure.** `getExposure` with the generator id and the exposure id, then store it
   alongside the image from the Modality API.

## Caveat worth stating to the user

`ExposureInfo` (Modality API) and `Exposure` (this API) describe the same physical event from two
sides and **share no schema**. Correlating them is your work, not the contract's. Neither is DICOM.
