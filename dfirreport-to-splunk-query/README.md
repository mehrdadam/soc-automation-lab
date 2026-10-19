
# From DFIR Reports to Splunk Queries

# AI-Assisted Detection Engineering with Splunk SOAR

[English](./README.md) | [فارسی](./README.fa.md)

---

## Project Overview

This project documents the architecture of a Splunk SOAR workflow that turns reports from The DFIR Report into candidate Splunk detection queries. The workflow monitors the [The DFIR Report](https://thedfirreport.com/) RSS feed, identifies new reports, and asks an AI model to generate suggested Splunk queries for each report. The results are stored in a Custom List for analysts to review and follow up on.

![AI Generated SPL Queries](./images/ai_spl.png)

The main goal is to shorten the gap between reading a threat report and preparing an initial draft for detection engineering. However, model-generated queries should be treated as initial suggestions, not as validated detection rules or queries ready for production use.

**This repository contains project documentation only. It does not include Python source code or an installable Splunk SOAR package.** It documents the architecture, configuration, data structures, and implementation details.

## How Does the Workflow Work?

![Splunk SOAR playbook overview](./images/Sc01.png)

To control how many recent posts are checked during the first playbook run, a Format block can be added at the beginning to set the desired number.

The next block is a Custom Function (CF) named `dfirreport_get_last_seen`. It retrieves the latest known URL and title and passes them to the next block.

The following CF, `dfirreport_check_dfirreport`, checks The DFIR Report website for recent posts. If new posts are found, it returns their URLs, titles, and the number of new posts.

A Decision block then checks whether the number of new posts is greater than zero. If it is, the results are passed to the next CF, `dfirreport_demux_posts`.

This block prepares multiple posts for individual processing. It allows the workflow to send requests to the AI model one at a time, rather than simultaneously, and parse each response separately.

The next CF, `dfirreport_build_prompt`, inserts the report title and URL into a predefined prompt. The prompt is structured as follows:

```python
prompt_text = (
    f"Threat report: {title}\n"
    f"URL: {url}\n\n"
    "IMPORTANT: Do NOT use any tools, do NOT run commands or fetch URLs. "
    "Answer directly from your own knowledge.\n"
    "Generate 3 Splunk detection queries for this threat. "
    'Return ONLY valid JSON, no explanation: '
    '{"splunk_queries": [{"name": "short name", "query": "index=* ..."}]}'
)
```

The prepared prompt is sent to the AI model through an HTTP action block configured to communicate with the AI API.

The response is then passed to `dfirreport_parse_ai_response`, which processes the model's output and extracts the requested queries.

Finally, `dfirreport_update_seen_row` stores the report title, URL, and generated queries in the Custom List.

From this point, additional action blocks can be added to email the queries using SMTP or notify a designated specialist using a Prompt block.

## Custom Function Reference

| Function | Purpose |
|---|---|
| `dfirreport_get_last_seen` | Reads the current state of the Custom List |
| `dfirreport_check_dfirreport` | Retrieves RSS data and identifies new reports |
| `dfirreport_demux_posts` | Separates reports so each report can be processed individually |
| `dfirreport_build_prompt` | Builds the prompt sent to the AI model |
| `dfirreport_parse_ai_response` | Processes the model response and extracts queries |
| `dfirreport_update_seen_row` | Records or updates information for a URL |

The function names match those used in the implementation. The function source code is not included in this repository.

## Custom List Structure

List name: `dfirreport_seen_posts`

| Column | Purpose |
|---|---|
| `url` | Report URL |
| `title` | Report title |
| `splunk_queries` | Suggested queries generated for the report |

The first row of the list is the header and must contain `url`, `title`, and `splunk_queries`. This row must be excluded from data processing when reading the list and preserved when rewriting it.

## Setup and Configuration

This section describes the configuration required to build the playbook. Since the function source code is not published, this documentation alone is not sufficient to install or run the project directly.

### 1. Prerequisites

- Splunk SOAR with the required permissions to create Custom Functions and playbooks
- An HTTP asset configured to communicate with an OpenAI API-compatible endpoint
- Network access to the model endpoint and a valid API key
- Access to The DFIR Report RSS feed
- A Custom List for storing report metadata and generated output

### 2. Create the Custom List

In Splunk SOAR, create a Custom List named `dfirreport_seen_posts` with the following columns:

- `url`
- `title`
- `splunk_queries`

The first row must contain these column names.

### 3. Configure the HTTP Asset

Create an HTTP asset for communication with the AI API.

- **Base URL:** The base URL of the AI API
- **Headers:**
  - `Authorization: Bearer <api-key>`
  - `Content-Type: application/json`

### 4. Scheduling

For automated execution, use an appropriate app, scheduler, or timer mechanism available in your Splunk SOAR environment.

## Technical Notes and Lessons Learned

### Model Responses Are Not Always Usable Text

Depending on the endpoint and model configuration, a response may contain `tool_calls` while the `content` field is empty.

If supported by the model and endpoint, setting `tool_choice: "none"` in the request can help prevent tool calls. The prompt should also explicitly instruct the model not to use tools.

## Security and Error Handling

- **Untrusted input:** Treat external report content as untrusted data. Report text must not be allowed to override the intended model instructions or bypass the defined constraints.
- **Query validation:** Before execution, validate SPL syntax, detection logic, search scope, and compatibility with the available telemetry.
- **Access control:** Restrict access to the HTTP asset, model endpoint, and Custom List according to operational requirements.
- **Communication failures:** Handle HTTP errors, timeouts, incomplete responses, invalid JSON, and responses without usable content separately. A failed request must not be recorded as successful query generation.
- **Output quality:** A generated query is not necessarily a correct query. Review its syntax, detection logic, data coverage, and performance before considering it for operational use.
- **Processing status:** Track report retrieval, model response processing, and output storage so that it is clear which stage each report has reached. Use appropriate logging for troubleshooting.
