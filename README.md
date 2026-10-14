# Multichannel API examples

Example requests for the [Multichannel API](openapi%20.yaml). Each file under `examples/` is [JSONC](https://code.visualstudio.com/docs/languages/json#_json-with-comments) (JSON with comments). The values are fictional and are there to show the shape of a request. Replace `jobType`, `customerId`, references, and file contents before sending anything.

Servers:

| Environment | Base URL |
| --- | --- |
| Test | `https://api-test.grafiskdigital.no` |
| Production | `https://api.grafiskdigital.no` |

## Authenticate

`POST /auth/login` with your username and API key. Send the returned JWT on later calls as `Authorization: Bearer <token>`.

```json
{ "username": "user@example.com", "apiKey": "your-api-key" }
```

`GET /me` returns the profile for the current token. `GET /customers` lists customers you may send for. `GET /info/job-types` lists job types, the schema each one accepts, and the channels it allows.

## Create a job

Post the body to the schema endpoint. `jobType` must be a job type from `/info/job-types` that is bound to that schema. `customerId` is the customer's organization number from `/customers`.

| Schema | Endpoint | Examples |
| --- | --- | --- |
| `invoice-v1` | `POST /jobs/invoice-v1` | [`examples/invoice-v1`](examples/invoice-v1) |
| `attachments-v1` | `POST /jobs/attachments-v1` | [`examples/attachments-v1`](examples/attachments-v1) |

A successful create returns `201` and a job summary (`id`, `status`, `documentCount`, …). `externalRef` on the job is checked so the same job is not created twice.

`channels` on the job applies to every document. `channels` on a document replaces the job list for that document. The list is tried in order. The first channel with a valid recipient is used. If no recipient is found, or the recipient is invalid for that channel, the next channel is tried. Allowed values: `email`, `sms`, `print`, `ehf`, `efaktura`, `digipost`, `avtalegiro`, `archive`, `neonomics`, `forward-billing`. Which of these a job type actually accepts is defined on the job type.

Set `onHold` to `true` to process the job without delivering it. If you omit it, the job type's own `onHold` setting is used.

### invoice-v1

Invoice jobs. `documentDetails.invoiceNumber` is required. Use this schema for EHF and for the other channels when the document is an invoice or credit note. `documentDetails.type` is `auto` (negative total becomes a credit note), `invoice`, or `credit-note`.

| Example | What it shows |
| --- | --- |
| [email.jsonc](examples/invoice-v1/email.jsonc) | Email with `templates.emailSubject` and an `emailTemplate` file |
| [sms.jsonc](examples/invoice-v1/sms.jsonc) | SMS with an inline text template |
| [print.jsonc](examples/invoice-v1/print.jsonc) | Print, with postal addresses on sender and recipient |
| [digipost.jsonc](examples/invoice-v1/digipost.jsonc) | Digipost. `ssn` is the identifier when set; otherwise the recipient fields must match one recipient |
| [efaktura.jsonc](examples/invoice-v1/efaktura.jsonc) | eFaktura invoice, same recipient lookup as Digipost |
| [ehf.jsonc](examples/invoice-v1/ehf.jsonc) | EHF to an organization, with invoice lines |
| [ehf-per-document-sender.jsonc](examples/invoice-v1/ehf-per-document-sender.jsonc) | Several legal entities in one job |
| [avtalegiro.jsonc](examples/invoice-v1/avtalegiro.jsonc) | AvtaleGiro with `ssn` and a PDF, and without either |
| [per-document-channels.jsonc](examples/invoice-v1/per-document-channels.jsonc) | One job, a different channel on each document |
| [multiple-channels.jsonc](examples/invoice-v1/multiple-channels.jsonc) | Channel fallback: eFaktura, Digipost, email, then print |

### attachments-v1

Jobs whose content is the attached files. `documentDetails` only carries `customFields`. Invoice lines and payment details belong on `invoice-v1`.

| Example | What it shows |
| --- | --- |
| [archive.jsonc](examples/attachments-v1/archive.jsonc) | Store a file on the archive channel |
| [with-attachment.jsonc](examples/attachments-v1/with-attachment.jsonc) | A main PDF plus a file with role `attachment` |
| [sms.jsonc](examples/attachments-v1/sms.jsonc) | SMS with an `smsTemplate` file and a PDF |
| [custom-fields.jsonc](examples/attachments-v1/custom-fields.jsonc) | Extra document metadata in `documentDetails.customFields` |
| [multiple-channels.jsonc](examples/attachments-v1/multiple-channels.jsonc) | Channel fallback: SMS, email, then print |

## Files

A document file is one of three objects:

- **Inline.** `{ "type": "base64", "mime": "application/pdf", "base64": "..." }`. `mime` is `application/pdf`, `text/plain`, or `text/html`. Use this on a single `POST /jobs/{schema}` for templates and small files.
- **Uploaded.** `{ "type": "file", "fileId": "<12 hex chars>" }`. Returned by `POST /jobs/{jobId}/files` on an open bulk job.
- **From a collection.** `{ "type": "collection", "collectionId": "<12 hex chars>", "filename": "invoice.pdf" }`. The collection is a zip or tar.gz uploaded with `POST /jobs/{jobId}/collections`.

`role` defaults to `main`. Other roles: `stationary`, `attachment`, `source`, `smsTemplate`, `emailTemplate`.

The examples send every file as `type: "base64"`. `"..."` stands in for the encoded bytes.

## Bulk jobs

For large batches, open a job, upload files, append documents, then commit:

1. `POST /jobs/{schema}/begin` with the job fields and no `documents`.
2. `POST /jobs/{jobId}/files` or `POST /jobs/{jobId}/collections` for the PDFs.
3. `POST /jobs/{jobId}/append` with `{ "documents": [ ... ] }`.
4. `POST /jobs/{jobId}/commit` with `{ "externalRefs": [ ... ] }` listing every document external ref you appended. The job is not committed if one is missing.

## Follow up

- `GET /jobs/{jobId}` and `GET /jobs` for job status.
- `GET /documents` to search, `GET /documents/{documentId}` for properties and logs, `GET /documents/{documentId}/pdf` for the PDF.
- `DELETE /jobs/{jobId}` to delete a job.
- `GET /info/job-statuses` and `GET /info/document-statuses` for the status names.
