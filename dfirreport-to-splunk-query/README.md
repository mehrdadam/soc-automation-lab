DFIR Report to Splunk Query

AI-Assisted Detection Engineering with Splunk SOAR

[English](./README.md) | [فارسی](./README.fa.md)




# Splunk SOAR — From DFIR Reports to Splunk Detection Queries

[English](./README.md) | [فارسی](./README.fa.md)

---

## Overview

This project documents a Splunk SOAR workflow that monitors [The DFIR Report](https://thedfirreport.com/), identifies new reports, and uses an AI model to generate candidate Splunk queries for each report. The results are stored in a SOAR Custom List for tracking and analyst review.

The idea is to shorten the path from threat research to an initial detection-engineering draft. The model output is a starting point, not a validated detection rule or a query ready for production use.

**This repository contains documentation only.** Python source files and an installable Splunk SOAR package are not included. The documentation covers the workflow, configuration, data structures, and technical issues encountered during implementation.

## Workflow

```text
The DFIR Report RSS feed
          │
          ▼
┌─────────────────────────┐
│ check_dfirreport        │
│ Fetch RSS and identify  │
│ new reports             │
└────────────┬────────────┘
             │ posts: [{url, title}, ...]
             ▼
┌─────────────────────────┐
│ demux_posts             │
│ Split reports for       │
│ individual processing   │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ build_prompt            │
│ Build the model request │
└────────────┬────────────┘
             │ body
             ▼
┌─────────────────────────┐
│ Native HTTP app in SOAR │
│ Send request to 9router │
│ /v1/chat/completions    │
└────────────┬────────────┘
             │ parsed_response_body
             ▼
┌─────────────────────────┐
│ parse_ai_response       │
│ Parse response and      │
│ extract candidate SPL   │
└────────────┬────────────┘
             │ splunk_queries
             ▼
┌─────────────────────────┐
│ update_seen_row         │
│ Store results in list   │
└─────────────────────────┘

parse_ai_response
          │
          ▼
     prompt block
     Display results for analyst review
```

`demux_posts` splits the list of reports into separate items so downstream processing can take place for each report. `build_prompt` constructs the request body, which the HTTP block sends to the model endpoint. `parse_ai_response` processes the response, and `update_seen_row` records the report details and candidate queries in the Custom List.

The final `prompt` block displays the formatted output for analyst review. The `get_last_seen` function is also intended to read the current list state at the beginning of a run.

## Custom Functions

| Function | Purpose | Inputs | Outputs |
|---|---|---|---|
| `dfirreport_check_dfirreport` | Fetch RSS and return new reports | `limit` | `status`, `posts`, `new_count` |
| `dfirreport_demux_posts` | Split the report list, producing one output per report | `posts` | `*.url`, `*.title` |
| `dfirreport_build_prompt` | Build the HTTP request body | `title`, `url` | `status`, `body` |
| `dfirreport_parse_ai_response` | Process the model response and extract queries | `ai_response`, `title`, `url` | `status`, `title`, `url`, `splunk_queries`, `formatted_output` |
| `dfirreport_update_seen_row` | Record or update the details for a URL | `url`, `title`, `splunk_queries` | `status`, `updated`, `count` |
| `dfirreport_get_last_seen` | Read the current Custom List state | — | `status`, `has_data`, `last_url`, `last_title`, `last_queries`, `count` |

The names reflect the implementation. The function source code is not published in this repository.

## Custom List

**Name:** `dfirreport_seen_posts`

| Column | Purpose |
|---|---|
| `url` | Report URL |
| `title` | Report title |
| `splunk_queries` | Candidate queries generated for the report |

The first row is the header and must contain `url`, `title`, and `splunk_queries`. Functions should skip this row when processing data and preserve it when rewriting the list.

`check_dfirreport` uses recorded URLs to identify reports that have already been seen. After processing, `update_seen_row` records or updates the report details and generated queries. Multiple queries in `splunk_queries` are separated by a blank line.

## Setup and configuration

This section describes the configuration needed to recreate the workflow. Since the Custom Function source code is not included, the README alone is not sufficient to install or run the project.

### 1. Prerequisites

- Splunk SOAR with permission to create Custom Functions and Playbooks
- An HTTP asset configured to reach an OpenAI-compatible API endpoint
- Network access to the model endpoint and a valid API key
- Access to The DFIR Report RSS feed
- A Custom List for tracking reports and storing results

This implementation uses [9router](https://github.com/decolua/9router) to route requests to a model.

### 2. Create the Custom List

Create a Custom List named `dfirreport_seen_posts` with these columns:

- `url`
- `title`
- `splunk_queries`

The first row should contain the column names.

### 3. Configure the Custom Functions

The workflow uses functions for RSS retrieval, report demultiplexing, request construction, response parsing, and Custom List updates.

To recreate the workflow, create the functions in SOAR and configure their input parameters and outputs according to the table above. Their source code is not available in this repository.

Implementation notes:

- The configured output type must match the data returned by each function. This implementation uses `item` and `list`.
- Helper functions are defined inside the main function body.
- Custom List writes in this implementation use the REST endpoint named `decided_list`, rather than `phantom.custom_list_add`.

### 4. Create the HTTP asset

Create an HTTP asset that can reach your 9router instance.

- **Base URL:** Your 9router address, for example `http://<host>:20128`
- **Headers:**
  - `Authorization: Bearer <api-key>`
  - `Content-Type: application/json`

The HTTP block sends:

`POST /v1/chat/completions`

The request body comes from the `body` output of `build_prompt`. Adjust the endpoint and authentication settings to match your deployment.

### 5. Connect the Playbook blocks

Connect the blocks according to the workflow diagram. The key data paths in this implementation are:

- `build_prompt.title` ← `demux_posts:custom_function_result.data.*.title`
- `build_prompt.url` ← `demux_posts:custom_function_result.data.*.url`
- HTTP body ← `build_prompt:custom_function_result.data.body`
- `parse_ai_response.ai_response` ← `post_data_N:action_result.data.*.parsed_response_body`
- `parse_ai_response.title` and `parse_ai_response.url` ← the corresponding `demux_posts` outputs
- `update_seen_row.splunk_queries` ← `parse_ai_response:custom_function_result.data.splunk_queries`
- Prompt block input ← `parse_ai_response:custom_function_result.data.formatted_output`

Replace `post_data_N` with the actual name of the HTTP block in your Playbook.

### 6. Scheduling

Add a Scheduler to the Playbook for automated execution. A daily schedule is one option; choose an interval that fits your operational requirements and environment.

## Technical notes and lessons learned

### 1. Connecting blocks does not automatically iterate over a list

If block A returns a list and connects to blocks B and C, the connections alone do not guarantee that B and C will run separately for every list element. The demux pattern creates a separate output for each element and enables one-to-many processing.

The `list_demux` function in the [phantomcyber/playbooks](https://github.com/phantomcyber/playbooks) repository is a useful reference for this pattern.

### 2. Datapaths may return lists rather than scalar values

A datapath such as `...data.*.url` may pass a value like:

`['https://example.com/report']`

rather than a scalar string:

`'https://example.com/report'`

If this distinction is not handled, a list-wrapped value may be stored in the Custom List and fail to match the original URL. Inspect function inputs and unwrap single-item lists at the boundary when necessary.

### 3. Model responses do not always contain usable text

With routed model endpoints, a response may contain `tool_calls` while the `content` field is empty. If supported by the selected model and endpoint, set `tool_choice: "none"` in the request body and explicitly instruct the model not to use tools.

Check `finish_reason` as well. A value of `length` indicates that generation reached a token limit. High `reasoning_tokens` usage may contribute to this in some models, but it does not establish the cause on its own. Review token limits and endpoint settings for the selected model.

### 4. Custom List updates in this implementation replace the entire list

The update method used here is:

1. Read the list using `GET decided_list/<name>`.
2. Modify the `content` array in memory.
3. Submit the full object using `POST`.

Preserve the header row, remove empty rows such as `['']`, and normalize list-wrapped values before rewriting the list. This normalization can also fix rows written with malformed values during earlier runs.

## Security and error handling

- **Untrusted input:** Treat external report content as untrusted data. Report text must not override the model's intended instructions or cause it to ignore the workflow's constraints.
- **Query validation:** Before execution, validate SPL syntax, detection logic, search scope, and compatibility with available telemetry.
- **API key protection:** Do not put API keys or authentication credentials in source code, screenshots, public documentation, or the GitHub repository.
- **Access control:** Restrict access to the HTTP asset, model endpoint, and Custom List according to operational requirements.
- **Communication and parsing errors:** Handle HTTP errors, timeouts, incomplete responses, invalid JSON, and responses without usable content. A failed request should not be recorded as successful query generation.
- **Output quality:** A generated query is not necessarily correct. Review its syntax, detection logic, data coverage, and performance before using it.
- **Processing status:** Check the RSS retrieval, model response, and Custom List update stages so that failures can be traced to the relevant step.

## Roadmap

- [x] RSS monitoring and identification of unrecorded URLs
- [x] Candidate query generation for each report through an HTTP block
- [x] Store report URLs, titles, and queries in a Custom List
- [x] Display formatted output for analyst review
- [ ] Run generated queries against real indexes and validate results
- [ ] Extract IOCs from report content
- [ ] Add an analyst approval workflow: `approve` / `reject` / `annotate`
- [ ] Add more feeds, including CISA and security vendor blogs
