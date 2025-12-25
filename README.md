# TestRail Migration Tool

A command-line tool for extracting data from [TestRail](https://www.testrail.com/) into a portable SQLite database. Perfect for migrating to QA Sphere, creating backups, or archiving test case data.

## Features

- Extract all projects, suites, sections, and test cases from TestRail
- Download all attachments (images, documents, etc.)
- Store everything in a single SQLite database file
- Compress the database using zstd for easy transfer
- Docker support for easy deployment

## Installation

### Download Pre-built Binaries

Download the latest release for your platform from the [Releases page](https://github.com/Hypersequent/testrail-migration-tool/releases):

| Platform | Architecture | Download |
|----------|-------------|----------|
| Linux | x86_64 | `testrail-migration-tool-linux-amd64` |
| Linux | ARM64 | `testrail-migration-tool-linux-arm64` |
| macOS | Intel | `testrail-migration-tool-darwin-amd64` |
| macOS | Apple Silicon | `testrail-migration-tool-darwin-arm64` |

```bash
# Example: Download and make executable on Linux x86_64
curl -LO https://github.com/Hypersequent/testrail-migration-tool/releases/latest/download/testrail-migration-tool-linux-amd64
chmod +x testrail-migration-tool-linux-amd64
mv testrail-migration-tool-linux-amd64 testrail-extractor
```

### Docker

```bash
docker pull ghcr.io/hypersequent/testrail-migration-tool:latest
```

## Configuration

Create a `.env` file with your TestRail credentials:

```bash
# TestRail API credentials
TR_USERNAME='your-email@example.com'
TR_PASSWORD='your-testrail-api-key'
TR_URL='https://your-org.testrail.io'
TR_COOKIE='your-testrail-session-cookie'

# Client name - used for database filename (alphanumeric only)
CLIENT_NAME='mycompany'
```

### Getting the Session Cookie

To download your attachments, you need to provide a session cookie:

1. Log in to TestRail in your browser
2. Open Developer Tools (F12)
3. Go to **Application** > **Cookies**
4. Copy the value of the `tr_session` cookie

## Usage

### Quick Start

```bash
# Create data directory
mkdir -p data

# Pull all data from active projects
./testrail-extractor pull_all

# Compress the database for easy transfer
./testrail-extractor compress_database
```

### Commands

#### `pull_all` - Extract Active Projects

Extracts all data from **active (non-archived)** projects:

```bash
./testrail-extractor pull_all
```

This command:
- Fetches all active projects from TestRail
- Downloads suites, sections, and test cases for each project
- Parses and downloads all attachments referenced in test cases
- Stores everything in `data/{CLIENT_NAME}.sqlite3`

#### `pull_all_completed` - Extract Archived Projects

Extracts all data from **completed (archived)** projects:

```bash
./testrail-extractor pull_all_completed
```

Use this if you need to migrate archived/completed projects.

#### `compress_database` - Compress for Transfer

Compresses the SQLite database using zstd compression:

```bash
./testrail-extractor compress_database
```

Output:
```
Compressing data/mycompany.sqlite3...

Compression Results:
  Input:       data/mycompany.sqlite3 (2.5 GB)
  Output:      data/mycompany.sqlite3.zst (450.2 MB)
  Ratio:       5.55:1
  Space saved: 82.0%
```

The compressed file is much smaller for transfer or archiving. SQLite databases typically compress very well (5-10x reduction).

### Docker Usage

Mount the `data` directory and pass environment variables:

```bash
# Create data directory
mkdir -p data

# Run with Docker
docker run --rm \
  -v "$(pwd)/data:/app/data" \
  -e TR_USERNAME='your-email@example.com' \
  -e TR_PASSWORD='your-api-key' \
  -e TR_URL='https://your-org.testrail.io' \
  -e CLIENT_NAME='mycompany' \
  ghcr.io/hypersequent/testrail-migration-tool:latest \
  pull_all
```

Or using an env file:

```bash
docker run --rm \
  -v "$(pwd)/data:/app/data" \
  --env-file .env \
  ghcr.io/hypersequent/testrail-migration-tool:latest \
  pull_all
```

## Output

The tool creates a SQLite database at `data/{CLIENT_NAME}.sqlite3` containing:

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

## Advanced Commands

For more granular control, these additional commands are available:

| Command | Description |
|---------|-------------|
| `list_projects` | List all available TestRail projects |
| `pull_project_all <project_id>` | Pull data for a specific project only |
| `pull_suites [project_id]` | Pull suites (optionally for specific project) |
| `pull_sections <project_id> <suite_id>` | Pull sections for a specific suite |
| `pull_cases <project_id> <suite_id>` | Pull test cases for a specific suite |
| `pull_attachments <project_id>` | Download attachments for a project |

## Troubleshooting

### "Failed to fetch projects"

- Verify your TestRail URL includes the protocol (`https://`)
- Check that your API key is correct
- Ensure your TestRail account has API access enabled

### Attachments not downloading

- Try adding the `TR_COOKIE` environment variable with your session cookie
- Some TestRail configurations restrict attachment access

### Large databases taking too long

- Use `pull_project_all <project_id>` to extract one project at a time
- Consider running overnight for very large TestRail instances

### Out of memory errors

- The tool streams data efficiently, but very large attachments may require more memory
- Try running with more available RAM or use Docker with increased memory limits

## License

MIT License - See [LICENSE](LICENSE) for details.

## Contributing

Issues and pull requests are welcome! Please open an issue first to discuss proposed changes.

## Related Projects

- [QA Sphere](https://qasphere.com) - Modern test management platform
- [TestRail](https://www.testrail.com/) - The source test management system
