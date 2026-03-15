# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A Go-based GitHub Action that pauses workflows and requires manual approval via GitHub Issues. Approvers comment on the issue with approval/denial keywords to resume or fail the workflow.

## Commands

```bash
# Run tests
go test -v .
# or
make test

# Build binary for release
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o manual-approval-x86_64-linux .

# Build Docker image (requires VERSION env var)
VERSION=1.x.x make build
VERSION=1.x.x make push

# Lint (runs in Docker)
make lint
```

## Architecture

The action runs as a compiled binary (not a Docker action). `action.yaml` downloads the binary from GitHub Releases, runs it, then reads `output.json` in a separate step.

**Flow:**
1. `main.go` validates env vars, creates GitHub client, retrieves approvers, creates issue, then blocks in a `select` waiting on either the comment poll loop or a kill signal
2. `approvers.go` — `retrieveApprovers()` resolves org teams to individual users via the GitHub Teams API; handles `exclude-workflow-initiator-as-approver`
3. `approval.go` — `approvalEnvironment` creates the GitHub issue; `approvalFromComments()` scans issue comments against the approvers list; returns `approvalStatusApproved`, `approvalStatusDenied`, or `approvalStatusPending`
4. `approvalstatus.go` — defines the `approvalStatus` type
5. `constants.go` — env var names and approval/denial keyword lists; custom words are appended to `approvedWords`/`deniedWords` at init time
6. On approval/denial, the binary writes `output.json` (contains `issueUrl`) before exiting — the third `action.yaml` step reads this file and sets the `issue_url` output

**Key design constraints:**
- The binary uses `exec` (not `os.Exit` directly in the run loop) to properly handle OS signals for graceful cancellation
- Org team approvers require a GitHub App token with read-only org member permissions (the default `GITHUB_TOKEN` lacks this permission)
- Token expiry: GitHub App tokens expire after 1 hour, capping approval wait time when using team approvers

## Release Process

1. `GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o manual-approval-x86_64-linux .`
2. Update the version in `action.yaml` (the `curl` download URL)
3. Create a GitHub release and attach the binary
