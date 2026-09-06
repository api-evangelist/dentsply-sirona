---
name: Acquire an intraoral image from a Dentsply Sirona sensor
description: >-
  Drive a Dentsply Sirona intraoral sensor through a full capture — find the device, open an
  acquisition session, arm it, watch the status stream, and retrieve the image — using the
  Intraoral Imaging Modality API.
api: openapi/dentsply-sirona-intraoral-modality-openapi.yml
generated: '2026-09-06'
method: generated
source: >-
  openapi/dentsply-sirona-intraoral-modality-openapi.yml,
  https://github.com/dsimaging/dsio-modality-api/wiki
operations:
  - getAllDevices
  - getDeviceInfo
  - getSensorInfo
  - createAcquisitionSession
  - subscribeAcquisitionStatus
  - setAcquisitionInfo
  - getAcquisitionStatus
  - getImages
  - getImage
  - getImageMedia
  - getImagePreview
  - deleteAcquisitionSession
---

# Acquire an intraoral image

This API drives **medical-device hardware that emits ionising radiation**. It does not fire the
X-ray source — a human presses the exposure button — but it arms the sensor and decides what gets
captured. Never run this flow unattended. See `agentic-access/dentsply-sirona-agentic-access.yml`
before wiring it to an agent.

## Before you start

- The service is **local**, not cloud. Base URL is `https://localhost:43809/api/dsio/modality/v1`
  on the workstation where the Dentsply Sirona Sensor Plugin service is installed.
- Auth is **HTTP Basic**. The username is yours; the password is an API key Dentsply Sirona issues
  directly to approved developers. There is no self-service signup.
  (`https://github.com/dsimaging/dsio-modality-api/wiki/Authentication`)
- There is **no idempotency contract**. Do not blind-retry `createAcquisitionSession` — a retry
  either opens a second session or returns `409 Device is in use`.
- Errors carry **no body**. The status code is the whole contract
  (`errors/dentsply-sirona-problem-types.yml`).

## Steps

1. **Find a device.** Call `getAllDevices`. Each `DeviceInfo` carries a `status` of `Available`,
   `InUse`, `Unavailable` or `Error`. Only `Available` can be used. Call `getDeviceInfo` for
   details on one, and `getSensorInfo` if you need sensor geometry or calibration.
2. **Open a session.** `createAcquisitionSession` with an `AcquisitionSessionInfo` body carrying
   `deviceId` and a `clientName` that identifies your software. Handle the declared failures
   explicitly: `404` device not found, `409` device already in use, `500` internal error,
   `507` not enough storage on the workstation. Keep the returned `sessionId`.
3. **Subscribe to status.** `subscribeAcquisitionStatus` holds an SSE connection open
   (`text/event-stream`) and pushes `AcquisitionStatus` as the exposure progresses. Use this rather
   than polling `getAcquisitionStatus`. `AcquisitionStatus.state` begins at `NoAcquisitionInfo`.
4. **Arm the sensor.** `setAcquisitionInfo` with the `AcquisitionInfo` for the exposure about to
   be taken — `enable: true`, plus `binning` and `applyLut` as your workflow requires.
   AcquisitionInfo must be supplied before **each** exposure. The state transitions to `Ready`
   when the sensor is armed. A `423 Locked` means the service is mid-read; wait for the state to
   leave the reading state and retry. A `400` means an invalid rotation value.
5. **Wait for the human.** The operator exposes the sensor. The status stream reports the
   transition and sets `lastImageId` when an image lands.
6. **Retrieve the image.** `getImages` lists the session's images; `getImage` returns one image's
   metadata; `getImageMedia` returns the full-resolution media and `getImagePreview` a preview.
   Media is `image/*` — PNG or TIFF, with a first-party `ImageInfo`/`ExposureInfo`/`LutInfo`
   envelope. **There is no DICOM here**; if your system speaks DICOM you must build the mapping.
7. **Close the session.** `deleteAcquisitionSession` releases the device. Do this even on the
   error paths, or the device stays `InUse` and the next `createAcquisitionSession` gets a `409`.
   Whether captured images survive session deletion is not documented — persist anything you need
   before you close.

## Reversibility

`deleteAcquisitionSession` reverses `createAcquisitionSession`. There is **no undo** for
`updateAcquisitionSession` or `setAcquisitionInfo`, and no reversal window is published for
anything. See `conventions/dentsply-sirona-conventions.yml`.
