# RHAI Jira access

The manual **Fullsend Jira connectivity** workflow reads `RHAI-13723` by default
and reports whether its issue type is `Epic`. It uses the same Fullsend commit
as the GitHub dispatcher. It does not poll, change Jira issues, or dispatch
agents. The ticket's type and hierarchy must be verified with an account that
can access it; they have not been verified as part of this configuration.

## Connect and inspect

Configure these Actions settings on `opendatahub-io/mlflow-go`:

| Setting | Kind | Value |
| --- | --- | --- |
| `JIRA_BASE_URL` | Variable | `https://redhat.atlassian.net` |
| `JIRA_USER_EMAIL` | Secret | Atlassian account email associated with the token |
| `JIRA_TOKEN` | Secret | Jira API token for that account |

The account needs Browse Projects access to RHAI and access through any
issue-security restrictions on the selected issue. The organization's policy
must permit API-token access. This read check does not require the project-role
administration or issue-property writes used by the poller.

After merging, run **Actions → Fullsend Jira connectivity → Run workflow** on
the default branch. Keep `RHAI-13723`, or supply another RHAI epic key. The job
summary confirms access and whether the type is `Epic`. A successful read of a
non-epic does not establish access to its parent epic. The workflow deliberately
omits titles, descriptions, comments, and raw API errors from public logs and
does not upload Jira content as an artifact.

For full issue content, use a trusted local terminal with the same environment
variables and a Fullsend CLI built at
`af8cd4756172ec105c3aa523515c9c7d51684e90`:

```bash
fullsend issues get --tracker jira --project RHAI --number 13723 \
  --jira-url "$JIRA_BASE_URL"
```

At this pin, the tracker constructs `RHAI-13723` from `--project RHAI` and
`--number 13723`; do not substitute Jira's separate internal numeric issue ID.
The JSON includes `issue_type`, title, description, labels, and comments.
It does not recursively load parents or epic children.

To inspect the hierarchy in Jira, open the issue's parent link. If API access
is needed, request `GET /rest/api/3/issue/RHAI-13723?fields=issuetype,parent`
on the configured Jira host. Read the returned parent's key and issue type;
a parent is not necessarily an epic. Repeat the parent lookup as needed.
For children of a confirmed epic, use Jira's enhanced search endpoint
`/rest/api/3/search/jql` with `project = RHAI AND parent = <confirmed-epic-key>`
and follow `nextPageToken` for every page. This selects direct children, not
all descendants. If the project uses a legacy Epic Link custom field, discover
its field ID from this tenant before using it; do not copy another tenant's
`customfield_*` ID.

Leave `.fullsend/config.yaml`'s tracker default unchanged. Explicit
`--tracker jira` selects Jira for these reads without changing GitHub issue
triage, coding, review, or fix behavior.

## What Jira-driven coding would additionally require

The [jira-code-test example](https://github.com/ralphbean/jira-code-test/blob/main/.github/workflows/fullsend-poll-jira.yaml)
uses a separate poller, a dispatch matrix, and custom code/triage harness
triggers. Its `FSENDAI` project and `type=Bug` query cannot discover RHAI epics.
Its moving workflow references and project-specific status transitions should
not be copied into this repository.

To extend this setup after the read check succeeds:

1. Add a repository-guarded poll workflow using the same pinned CLI and reusable
   dispatcher. Pass `--jira-project RHAI` even with a custom `--jql`, so Fullsend
   can resolve project roles for slash-command authorization. Start with
   `project = RHAI AND key = RHAI-13723`; use
   `project = RHAI AND issuetype = Epic` only when project-wide epic discovery
   is intended. Selecting an epic does not select its children.
2. Register local code/triage harness overrides through `agents[].source` in
   `.fullsend/config.yaml`. Add Jira work-item CEL triggers and retain the
   configured Pi runtime and OpenAI models. A Jira URL or `tracker: jira` alone
   does not enable polling or triggers.
3. Forward the poller's matrix, `jira_base_url`, Jira credentials, and the
   existing mint/inference settings to the reusable dispatcher. Validate RHAI's
   actual role names and workflow transitions rather than copying FSENDAI's
   `Closed` transition overrides.
4. Resolve the pinned built-in agents' Jira payload limitation before enabling
   dispatch. The [pinned integration guide](https://github.com/fullsend-ai/fullsend/blob/af8cd4756172ec105c3aa523515c9c7d51684e90/docs/guides/user/jira-integration.md)
   says their pre/post scripts still expect GitHub issue numbers and that
   substantive results are not automatically posted back to Jira. A successful
   poll or issue read does not prove Jira-to-PR coding works.

Unlike this read check, polling writes locks and checkpoints to Jira issue
properties. The pinned poller documents Developers and Administrators project
roles for its service account, and maps command authors' Jira role names to
authorization without checking GitHub repository membership. Confirm this with
the RHAI administrators before activation. Do not run a poll merely as a
read-only connection test.

## Troubleshooting

- A failed read can mean an expired token, an incorrect account email, missing
  project/issue access, or an organization policy blocking API tokens. Reproduce
  locally to inspect the error without publishing private Jira response bodies.
- An accessible issue with a non-Epic type requires a separate parent lookup;
  the connectivity workflow does not infer an epic from the issue key.
- An empty poll dispatch list can mean no eligible recent activity, even when
  the issue is visible. The pinned poller limits first-run backfill to 24 hours.

References:

- [Pinned issue CLI](https://github.com/fullsend-ai/fullsend/blob/af8cd4756172ec105c3aa523515c9c7d51684e90/internal/cli/issues.go)
- [Pinned Jira key construction](https://github.com/fullsend-ai/fullsend/blob/af8cd4756172ec105c3aa523515c9c7d51684e90/internal/tracker/jira_client.go)
- [Jira issue REST API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/)
- [Jira enhanced search API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-search/)
