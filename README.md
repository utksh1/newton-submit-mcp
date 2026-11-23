# newton-submit-mcp

Model Context Protocol server for the Newton School platform. It exposes course,
assignment, playground, GitLab, and evaluation-runner operations as tools that any
MCP-compatible agent can call over stdio.

For agent-specific instructions and verified API workflows, see [AGENTS.md](./AGENTS.md).

## Requirements

- Node.js 20 or newer
- npm
- A Newton School student access token

## Installation

```bash
git clone https://github.com/utksh1/newton-submit-mcp.git
cd newton-submit-mcp
npm install
npm run build
```

The build compiles `src/index.ts` into `build/index.js` and marks it executable.
Run `node build/index.js` to start the server (it communicates over stdio).

## Authentication

Create `~/.newton-submit-mcp/auth.json` with mode `0600`:

```json
{ "token": "<your Newton School access token>" }
```

```bash
mkdir -p ~/.newton-submit-mcp
chmod 700 ~/.newton-submit-mcp
chmod 600 ~/.newton-submit-mcp/auth.json
```

The token is auto-loaded at startup, so it does not need to appear in any MCP
host configuration. You can also set it at runtime with the `set_auth_token`
tool; that persists to the same file.

Never commit the token or paste it into documentation or chat.

## MCP host configuration

Generic stdio configuration:

```json
{
  "mcpServers": {
    "newton-submit": {
      "command": "node",
      "args": ["/absolute/path/to/newton-submit-mcp/build/index.js"]
    }
  }
}
```

Some MCP clients use a different registration command; consult your client's
documentation. Point it at `build/index.js` using an absolute path, then restart
the client so it discovers the tools. Tool names may be prefixed with the
configured server name.

## Tools

### Authentication and profile

| Tool | Description | Required arguments |
| --- | --- | --- |
| `set_auth_token` | Set and persist the Newton School token | `token` |
| `get_user_info` | Fetch the current user profile | — |

### Courses and assignments

| Tool | Description | Required arguments |
| --- | --- | --- |
| `list_courses` | List enrolled courses | — |
| `list_assignments` | List assignments for a course, optionally filtered by title | `course_hash` |
| `list_questions` | List questions in an assignment | `assignment_hash` |
| `list_assessments` | List quizzes and assessments for a course | `course_hash` |
| `get_assessment_questions` | List questions in an assessment | `course_hash`, `assessment_hash` |

### Code and repositories

| Tool | Description | Required arguments |
| --- | --- | --- |
| `get_project` | Get playground configuration and GitLab credentials | `project_id` (playground hash) |
| `get_file` | Read a file from a GitLab repository | `gitlab_token`, `project_id` (numeric), `file_path` |
| `update_file` | Create or update a file in GitLab | `gitlab_token`, `project_id`, `file_path`, `content` |

### Submission and evaluation

| Tool | Description | Required arguments |
| --- | --- | --- |
| `submit_solution` | Trigger evaluation for a playground project or coding question | `project_id` (playground hash) |
| `wait_for_submission` | Poll until a build finishes and return test results | `project_id`, `username` |
| `get_submission_status` | Get the latest submission status and test cases | `project_id`, `username` |
| `list_all_submissions` | List submission history for a playground | `project_id` |

All tools that accept `access_token` use the persisted token when it is omitted.

`submit_solution` expects a playground hash, not an assignment or question hash.
See [AGENTS.md](./AGENTS.md#5-submission-workflows--verified) for the verified
endpoint chain that resolves question hashes into playground hashes and for the
subjective and Newton Box submission routes.

## Development

```bash
npm install
npm run build      # compile TypeScript and chmod build/index.js
npm start          # run the built server
npm run watch      # tsc --watch
```

`npm run build` is also the `prepare` script, so it runs automatically after
installation. Verify a change with a fresh build and one live API call (for
example `get_user_info`) before considering it done.

## Security notes

- The token is a student credential: store it only in `~/.newton-submit-mcp/auth.json`
  with mode `0600` and never log it.
- The MCP subprocess inherits a filtered environment; pass secrets explicitly
  only if a host requires it. This server does not require an environment token.
- Generated solution files and scratch data belong outside the repository.

## License

MIT — see [LICENSE](./LICENSE).
