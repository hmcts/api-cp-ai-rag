# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Shared team standards (commands, generator settings, CI standard set, publishing) are
live-imported from `hmcts-apim-sdlc-orchestrator` via `.claude/CLAUDE.md` — only
repo-unique content lives here.

## Repo: api-cp-ai-rag

OpenAPI contract for the RAG (Retrieval Augmented Generation) service — document ingestion into AI search and LLM answer retrieval over the ingested documents.

**Pattern**: Pure spec-only
**OpenAPI spec version**: 0.0.0 in-repo (`openapi: 3.0.0`; version injected by CI)
**OpenAPI Generator version**: 7.16.0
**Spring Boot version**: 4.0.0-M3

## API Endpoint(s)

- `POST /document-upload` — initiate upload, returns storage URL — 200, 400, 500
- `GET /document-upload/{documentReference}` — document ingestion status — 200, 400, 404, 500
- `POST /answer-user-query` — synchronous answer — 200, 400, 500
- `POST /answer-user-query-async` — accept async answer request — 202, 400, 500
- `GET /answer-user-query-async-status/{transactionId}?withChunkedEntries=` — async answer status/result — 200, 400, 404, 500

All error responses use the shared `requestErrored` schema (`errorMessage` only).

## Generated Interfaces & Schema

- Spec file: `src/main/resources/openapi/ai-rag-service.openapi.yml` (must be the **only** `*.openapi.yml` in that directory — build resolves via `fileTree(...).singleFile`)
- Schema dir: `src/main/resources/openapi/schema/` contains only `document-status.example.json` — no paired `*.schema.json`, and the spec does not `$ref` it
- Generated API interfaces (`uk.gov.hmcts.cp.openapi.api`, one per tag):
  - `DocumentIngestionInitiationApi` — POST /document-upload
  - `DocumentIngestionStatusApi` — GET /document-upload/{documentReference}
  - `DocumentInformationSummarisedSynchronouslyApi` — POST /answer-user-query
  - `DocumentInformationSummarisedAsynchronouslyApi` — async pair of endpoints

## Domain Models

| Model | Purpose |
|---|---|
| `DocumentUploadRequest` | Client-supplied document id/name, metadata filters, optional soft-delete overwrites |
| `FileStorageLocationReturnedSuccessfully` | Storage URL for the client to upload the document to |
| `DocumentIngestionStatus` | Ingestion status enum — `METADATA_VALIDATED`/`INVALID_METADATA` are deprecated, retained for backwards compatibility |
| `DocumentIngestionStatusReturnedSuccessfully` | Status lookup response |
| `AnswerUserQueryRequest` | User query plus metadata filters |
| `UserQueryAnswerReturnedSuccessfullySynchronously` | Sync answer with source `DocumentChunk`s |
| `UserQueryAnswerRequestAccepted` | 202 body with `transactionId` for async polling |
| `UserQueryAnswerReturnedSuccessfullyAsynchronously` | Async answer with `AnswerGenerationStatus`, optional chunks |
| `DocumentChunk` / `MetadataFilter` | Chunk provenance for answers; key/value metadata filter |
| `RequestErrored` | Shared error body (`errorMessage`) |

## Test Structure

| Class | What it validates |
|---|---|
| `OpenAPIConfigurationLoaderTest` | Spec on the classpath parses via `OpenAPIV3Parser` (uses the only hand-written class, `OpenAPIConfigurationLoader`) |

## Generator Config Notes

- Missing `@JsonInclude(NON_NULL)` in `additionalModelTypeAnnotations` — null fields will appear in JSON responses
- Uses deprecated `inputSpec =` assignment — migrate to `inputSpec.set(layout.projectDirectory.file(...))`
- OpenAPI Generator 7.16.0 — target 7.22.0 per upgrade cycle
- Spring Boot 4.0.0-M3 — target 4.0.6+ per upgrade cycle
- No `gradle/*.gradle` config modules — all build logic is in the single `build.gradle`
- Spec is OpenAPI 3.0.0 (3.1.0 preferred for new specs; fine for existing)

## CI/CD Deviations

- SwaggerHub/APIHub publishing removed (no `publish-openapi-spec.yml`); instead `publish-api-docs.yml` publishes Swagger UI to https://hmcts.github.io/api-cp-ai-rag/ via `hmcts/amp-catalog` on release
- No `auto-merge-dependabot.yml`
- No AJV schema-vs-example validation in `lint-openapi.yml` (no JSON schemas to validate)

## Repo-Specific Notes

- Backwards compatibility is contract policy: deprecated `documentIngestionStatus` enum values are retained, not deleted; `GET /document-status` was removed only alongside deprecating its Flow B enum values
- Commit history mixes Jira-prefixed (`DD-NNNNN: ...`) and Conventional Commits styles; follow Conventional Commits with the Jira ticket in the footer per hmcts-standards
- `OpenAPIConfigurationLoader` hardcodes the spec path (`openapi/ai-rag-service.openapi.yml`) — renaming the spec file requires updating it
