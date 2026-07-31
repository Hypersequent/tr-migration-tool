# TR Migration Tool

A command-line tool for extracting data from TestRail into a portable SQLite database. Perfect for migrating to [QA Sphere](https://qasphere.com/), creating backups, or archiving test case data.

## How to Use

The tool is distributed as a Docker container. This ensures isolation from the host system and compatibility across platforms. You can run it on **any machine with Docker** that has **network access to your TestRail URL** — including your local workstation.

## Preparation

Create an empty directory on your system and navigate into it:

```bash
mkdir tr-to-qas
cd tr-to-qas
mkdir data
```

Ensure you can pull the Docker image:

```bash
docker pull ghcr.io/hypersequent/tr-migration-tool:latest
```

If not, you need to [authenticate with GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-with-a-personal-access-token-classic) using any GitHub account.

## Configuration

Create an `env.txt` file with your TestRail credentials:

```bash
# TestRail API credentials
TR_USERNAME=your-email@example.com
TR_PASSWORD=your-testrail-api-key
TR_URL=https://your-org.testrail.io
TR_COOKIE=your-testrail-session-cookie

# Client name - used for database filename (alphanumeric only)
CLIENT_NAME=yourorg
```

**Note:** `TR_URL` must be the external TestRail URL accessible from your browser, not an internal or Docker container address.

**Warning:** This file contains sensitive credentials. Delete it after completing the migration.

### Getting the Session Cookie

To download attachments, you need to provide a session cookie:

1. Log in to TestRail in your browser
2. Open Developer Tools (F12)
3. Go to **Application** > **Cookies**
4. Copy the value of the `tr_session` cookie

### Verify API Access

Run the following command to check that the TestRail API is working:

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$(pwd)/data:/app/data" \
  --env-file env.txt \
  ghcr.io/hypersequent/tr-migration-tool:latest \
  list_projects
```

If everything is configured correctly, you should see a list of your TestRail projects, one
per line, with their id, name and status:

```
12	Website Regression	[active]
13	Mobile App	[active]
14	Legacy Suite	[inactive]
```

### Pull Data

Once verified, launch the tool to pull your data:

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$(pwd)/data:/app/data" \
  --env-file env.txt \
  ghcr.io/hypersequent/tr-migration-tool:latest \
  pull
```

By default (no flags) this command:
- Fetches all **active** projects from TestRail
- Downloads suites, sections, and test cases for each project
- Parses and downloads all attachments referenced in test cases
- Stores everything in `data/{CLIENT_NAME}.sqlite3`
- Compresses the result automatically at the end

Depending on the size of your TestRail installation, this command can run from minutes to
hours.

Once complete, transmit the compressed file `data/{CLIENT_NAME}.sqlite3.zst` to the QA Sphere team for importing.

#### Choosing which projects to pull

Add flags after `pull` to control which projects are included:

| Flag | Pulls |
| --- | --- |
| *(none)* | Active projects only (the default) |
| `--all` | Every project, active and archived |
| `--active` | Active projects only |
| `--inactive` | Archived (completed) projects only |
| `--project <id>` | One specific project by id; repeat the flag to add more |

Flags combine, so `pull --active --project 12 --project 15` pulls every active project
*plus* projects `12` and `15`, even if those two are archived. Project ids come from
`list_projects`.

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$(pwd)/data:/app/data" \
  --env-file env.txt \
  ghcr.io/hypersequent/tr-migration-tool:latest \
  pull --active --project 12 --project 15
```

#### Running it more than once

It's safe to run `pull` multiple times, including with a different selection each time —
new data is added to the existing database, not overwritten, and anything already pulled is
updated in place rather than duplicated. This means you can, for example, pull your active
projects first and pull a couple of archived ones later, all into the same
`data/{CLIENT_NAME}.sqlite3`. Each run recompresses the whole file, so the `.zst` always
reflects everything pulled so far.

If you need to re-run compression manually (e.g. after inspecting the database yourself):

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$(pwd)/data:/app/data" \
  --env-file env.txt \
  ghcr.io/hypersequent/tr-migration-tool:latest \
  compress_database
```

## Other Commands

#### `pull_projects` - Refresh the Project List

`list_projects` and `pull` already fetch the project list for you the first time you run
either of them against an empty database, so this is rarely needed. Run it directly only to
pick up a project created in TestRail *after* your first pull — otherwise the tool won't
know it exists.

#### `warnings` - Review Issues Found During a Pull

Prints anything that went wrong during the last `pull` (e.g. an attachment that failed to
download), grouped by type. Worth checking after a large pull, especially if it took an
unusually long time or you saw errors scroll by.

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$(pwd)/data:/app/data" \
  --env-file env.txt \
  ghcr.io/hypersequent/tr-migration-tool:latest \
  warnings
```

## What the SQLite File Contains

The tool creates a SQLite database at `data/{CLIENT_NAME}.sqlite3` containing your TestRail project structure and test case data. This includes all projects with their metadata, test suites, section hierarchy (folder structure), and complete test cases with all fields. The database also stores downloaded attachments as binary data, along with reference tables for case types, custom field definitions, and priority levels.

| Table | Contents |
|-------|----------|
| `projects` | Project metadata (ID, name, suite mode, etc.) |
| `suites` | Test suites |
| `sections` | Folder structure within suites |
| `cases` | Test cases with all fields |
| `attachments` | Downloaded attachment files (as BLOBs) |
| `case_types` | TestRail case type definitions |
| `case_fields` | Custom field definitions |
| `priorities` | Priority level definitions |

The SQLite file does not contain user lists or any test run content.

## Troubleshooting

### "Failed to fetch projects"

- Verify your TestRail URL includes the protocol (`https://`)
- Check that your API key is correct
- Ensure your TestRail account has API access enabled

### Attachments not downloading

- Try adding the `TR_COOKIE` environment variable with your session cookie
- Some TestRail configurations restrict attachment access

### 500 errors during attachment download

- TestRail may return 500 errors for some attachments due to internal consistency issues
- These errors are usually tolerable and the migration will continue with other attachments

### Large databases taking too long

- Use `pull --project <project_id>` to extract one project at a time
- Consider running overnight for very large TestRail instances

### Out of memory errors

- The tool streams data efficiently, but very large attachments may require more memory
- Try running with more available RAM or use Docker with increased memory limits

### Still having issues?

The tool automatically saves logs to `data/log-{datetime}.txt` (e.g., `log-20251225-143000.txt`). If you're unable to resolve the problem, please send the relevant log file to the QA Sphere support team for assistance.

## Contributing

Feedback is welcome. Please contact QA Sphere support for troubleshooting.

## Related Links

- [QA Sphere](https://qasphere.com) - Modern test management platform
