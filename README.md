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
TR_USERNAME='your-email@example.com'
TR_PASSWORD='your-testrail-api-key'
TR_URL='https://your-org.testrail.io'
TR_COOKIE='your-testrail-session-cookie'

# Client name - used for database filename (alphanumeric only)
CLIENT_NAME='yourorg'
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

If everything is configured correctly, you should see a list of your TestRail projects.

### Pull Data

Once verified, launch the tool to pull all data:

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$(pwd)/data:/app/data" \
  --env-file env.txt \
  ghcr.io/hypersequent/tr-migration-tool:latest \
  pull_all
```

This command:
- Fetches all active projects from TestRail
- Downloads suites, sections, and test cases for each project
- Parses and downloads all attachments referenced in test cases
- Stores everything in `data/{CLIENT_NAME}.sqlite3`

Depending on the size of your TestRail installation, this command can run from minutes to hours. Compression runs automatically at the end.

Once complete, transmit the compressed file `data/{CLIENT_NAME}.sqlite3.zst` to the QA Sphere team for importing.

If you need to re-run compression manually:

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$(pwd)/data:/app/data" \
  --env-file env.txt \
  ghcr.io/hypersequent/tr-migration-tool:latest \
  compress
```

## Other Commands

#### `pull_all_completed` - Extract Archived Projects

Fetches only completed (archived) projects. If you need both active and archived projects, run `pull_all` first, then `pull_all_completed`.

#### `pull_project_all <project_id>` - Pull Specific Project

Pull data for a specific project only. Useful when you need to extract a single project.

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

- Use `pull_project_all <project_id>` to extract one project at a time
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
