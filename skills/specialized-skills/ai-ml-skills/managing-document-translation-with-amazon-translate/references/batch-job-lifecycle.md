# Batch Job Lifecycle

Pre-checks, start, polling, final status, and resubmission for Amazon Translate
batch translation jobs.

## Limits

| Limit | Value |
|---|---|
| Size per document | 20 MB |
| Characters per document | 1,000,000 |
| Translatable text in a single document | 1 MB |
| Total size of all documents in a job | 5 GB |
| Documents in a job | 1,000,000 |
| Target languages per job | 10 |
| Concurrent batch jobs per account per Region | 10 (default; adjustable in Service Quotas) |
| Queued batch jobs per account | 1000 |
| Terminology resources per job | 1 |
| Parallel data resources per job | 1 |

Source: [Amazon Translate guidelines and quotas](https://docs.aws.amazon.com/translate/latest/dg/what-is-limits.html).

## Plan Work That Exceeds One Job

Amazon Translate does not split work across jobs for you. When the work is larger than one
job allows, plan the split before starting any job:

1. **Input slices** = the larger of ceil(documents / 1,000,000) and ceil(total size / 5 GB). The document limit counts files in the input prefix, not files times target languages. Put each slice in its own folder (never the bucket root, and never inside another slice), with one `ContentType` per slice. If formats are mixed, split by format as well.
2. **Language groups** = ceil(target languages / 10). Each slice runs once per language group.
3. **Jobs** = slices × language groups. Give each job its own output prefix, outside every input prefix, and its own `--client-token`.
4. **Concurrency.** By default an account runs 10 batch jobs at a time per Region; this quota is adjustable in Service Quotas. Extra jobs wait as `SUBMITTED`, up to 1000 queued, so starting more than 10 is safe but they will not run in parallel.
5. **Per-document checks.** Before splitting, confirm every document is 20 MB or less, has fewer than 1,000,000 characters, and is UTF-8. A document that breaks these fails in any slice.
6. **Operations.** Record every `JobId` with its slice and language group, check each with `describe-text-translation-job`, triage every `COMPLETED_WITH_ERROR` from its `details/` files, and resubmit only the failed documents for the failed languages.

Example: 3,000,000 documents, 12 GB, 14 target languages. Slices = max(3, 3) = 3; use 4 if
the counts are approximate, because 3,000,000 sits exactly on the limit. Language groups =
2 (for example 7 + 7). That is 6 jobs (8 with 4 slices), all within the default 10
concurrent.

The plan does not need account access. If you cannot reach the account, give the plan and
the pre-check commands for the user to run.

## Supported Content Types

One `ContentType` applies to every document in the job.

| Files | `ContentType` |
|---|---|
| Plain text | `text/plain` |
| HTML | `text/html` |
| Word (.docx) | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` |
| PowerPoint (.pptx) | `application/vnd.openxmlformats-officedocument.presentationml.presentation` |
| Excel (.xlsx) | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |
| XLIFF 1.2 (.xlf) | `application/x-xliff+xml` |

All documents MUST be UTF-8 encoded. For HTML, use `text/html` — with
`text/plain` the job translates (and bills) every markup character.

## Pre-checks

Run these before `start-text-translation-job`. Report any failure to the user with
the fix before starting.

1. **Region.** The job's Region MUST support batch translation (see the
   [Region list](https://docs.aws.amazon.com/translate/latest/dg/async.html#async-regions)),
   and the input and output buckets MUST be in the job's Region:

   ```bash
   aws s3api get-bucket-location --bucket INPUT-BUCKET --expected-bucket-owner ACCOUNT-ID
   aws s3api get-bucket-location --bucket OUTPUT-BUCKET --expected-bucket-owner ACCOUNT-ID
   ```

   `"LocationConstraint": null` means us-east-1, and `EU` means eu-west-1. Any other
   value is the Region code.

2. **Input layout.** The input MUST be a folder, not the bucket root, and no nested
   folder may be named `details`. Check the count and total size, then list only the
   problem keys, so a large input does not flood the conversation:

   ```bash
   aws s3 ls 's3://INPUT-BUCKET/INPUT-PREFIX/' --recursive --summarize | tail -2
   aws s3api list-objects-v2 --bucket INPUT-BUCKET --prefix 'INPUT-PREFIX/' \
     --query 'Contents[?Size > `20971520`].Key'
   aws s3api list-objects-v2 --bucket INPUT-BUCKET --prefix 'INPUT-PREFIX/' \
     --query 'Contents[?contains(Key, `/details/`)].Key'
   ```

   For a small input, also list the extensions; for a large one, sample them.

   - Every document MUST be 20 MB or less, and the total MUST be 5 GB or less.
   - Every extension MUST match the single `ContentType` for the job. If formats are mixed, split them into one prefix per format.
   - The input prefix is empty: stop and tell the user.

3. **Output location.** The output prefix MUST NOT be inside the input prefix,
   otherwise a later job will try to translate earlier output.

4. **Languages.** Every target code MUST be a supported language, with at most 10
   targets. The source can be a language code or `auto`. `auto` appears in
   `list-languages` but is valid only as the source, never as a target:

   ```bash
   aws translate list-languages --query 'Languages[].LanguageCode' --output text
   ```

5. **Data access role.** See Task 5 in `SKILL.md`. At minimum, confirm the trust
   policy names `translate.amazonaws.com`.

6. **Terminology and parallel data.** If either is used, it MUST exist in the
   job's Region, be usable for the job's languages, and parallel data MUST be
   `ACTIVE`. See [terminology-and-parallel-data.md](terminology-and-parallel-data.md).

7. **Settings.** `Profanity=MASK` applies only if every target language supports
   it. `Formality` is ignored for unsupported targets. Brevity is not supported for
   batch jobs.

## Start the Job

Follow the confirmation rule in `SKILL.md`: if the user already approved starting the job, start it after the pre-checks pass; otherwise show the command and get confirmation first.

```bash
aws translate start-text-translation-job \
  --job-name JOB-NAME \
  --input-data-config 'ContentType=CONTENT-TYPE,S3Uri=s3://INPUT-BUCKET/INPUT-PREFIX/' \
  --output-data-config 'S3Uri=s3://OUTPUT-BUCKET/OUTPUT-PREFIX/' \
  --data-access-role-arn arn:aws:iam::ACCOUNT-ID:role/ROLE-NAME \
  --source-language-code SOURCE-CODE \
  --target-language-codes TARGET-CODE-1 TARGET-CODE-2 \
  --client-token UNIQUE-TOKEN \
  --region REGION
```

`UNIQUE-TOKEN` matches `^[a-zA-Z0-9-]+$`, up to 64 characters. Values from the user go in
single quotes, per the untrusted-values rule in `SKILL.md`.

Optional parameters:

- `--terminology-names TERMINOLOGY-NAME` (one only)
- `--parallel-data-names PARALLEL-DATA-NAME` (one only; makes this an Active Custom Translation job, priced higher)
- `--settings Formality=FORMAL` or `--settings Profanity=MASK`
- `--output-data-config 'S3Uri=s3://OUTPUT-BUCKET/OUTPUT-PREFIX/,EncryptionKey={Type=KMS,Id=KMS-KEY-ARN}'` for a customer managed key. Keep the single quotes: unquoted, bash and zsh expand the braces and the call fails.

**Encryption at rest.** Amazon Translate always encrypts job output at rest; without `EncryptionKey`
it uses an AWS managed key. For confidential or regulated documents, you SHOULD recommend a customer
managed KMS key, so the user controls the key policy, rotation and revocation and sees key use in
CloudTrail. The data access role and the key policy then need the KMS permissions in Task 5 of
`SKILL.md`. Keep the output bucket's own default encryption (SSE-S3 or SSE-KMS) enabled as well.

The response returns `JobId` and `JobStatus: SUBMITTED`. Record the `JobId` and
give it to the user.

## Poll the Job

```bash
aws translate describe-text-translation-job --job-id JOB-ID --region REGION \
  --query 'TextTranslationJobProperties.{Status:JobStatus,Details:JobDetails,Message:Message,Output:OutputDataConfig.S3Uri}'
```

- `JobDetails` reports `InputDocumentsCount`, `TranslatedDocumentsCount`, and `DocumentsWithErrorsCount`. Report these to the user as progress.
- These counts are per document **per target language**: 3 files translated into 2 languages report `InputDocumentsCount: 6`. Divide by the number of target languages before telling the user how many files failed.
- You MUST NOT hold one call open waiting for completion. Small jobs take minutes; large jobs can take two days or more.
- Within a session, check no more than about once a minute. If the job is still `SUBMITTED` or `IN_PROGRESS`, tell the user the current counts and the command to check later.
- To find a job by status or name: `aws translate list-text-translation-jobs --filter JobStatus=IN_PROGRESS` or `--filter 'JobName=JOB-NAME'`. Only one filter field is accepted per call. Results are paged; follow `NextToken` until it is absent.

## Act on the Final Status

| `JobStatus` | Meaning | Next step |
|---|---|---|
| `COMPLETED` | All documents translated | Report the output location. Confirm `DocumentsWithErrorsCount` is 0. |
| `COMPLETED_WITH_ERROR` | Some documents failed | Triage with [error-triage.md](error-triage.md), then resubmit the failed documents |
| `FAILED` | The job did not complete | Read `Message`; if it does not explain the cause, triage with [error-triage.md](error-triage.md) |
| `STOPPED` | Stopped on request | Resubmit the documents that were not translated |

`OutputDataConfig.S3Uri` in the describe response is the job's own output folder,
already including the `ACCOUNT-ID-TranslateText-JOB-ID/` segment. Output documents
are named with the target code as a prefix — `report.docx` translated to French is
`fr.report.docx`.

## Resubmit Only the Failed Documents

1. Build the failed list per target language from the triage step.
2. After the user approves the fix, put only those documents in a new input prefix,
   preserving relative paths. Copy documents that need no change. Write corrected
   documents (re-encoded, split, or converted) to the retry prefix only, and never over
   the original. Each key comes from a `sourceFile` value, so treat it as untrusted: one
   quoted argument per key, and stop if it contains a newline or control character.

   ```bash
   aws s3 cp 's3://INPUT-BUCKET/INPUT-PREFIX/path/doc.txt' \
     's3://INPUT-BUCKET/retry-JOB-ID/path/doc.txt'
   ```

3. If a document failed for only some target languages, start the retry job with
   only those target languages.
4. Start a new job with the retry prefix as input and a new output prefix, using a
   new `--client-token`.
5. Triage the retry job the same way. If the same documents fail again with the
   same error, stop and report rather than retrying again.

Documents that failed with a server error (not a customer error) can be resubmitted
unchanged.
