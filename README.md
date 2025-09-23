# release-please-poc

## 💬 Local Commit Message Linting (Husky + Commitlint)

To ensure a consistent Git history and enable automated versioning, this project enforces a local commit message validation system.

### Goal

This system solves two primary objectives:
1.  **Error Prevention:** It blocks commits with improperly formatted messages before they are even created in the repository.
2.  **Release Automation:** It guarantees that all commits merged into the main branch adhere to the [Conventional Commits](https://www.conventionalcommits.org/) standard, which our `release-please` tool relies on.

### How It Works

We use two main utilities:
-   **Husky**: Manages Git hooks. We use the `commit-msg` hook, which triggers every time you run `git commit`.
-   **Commitlint**: Validates the commit message against a defined set of rules.

The process is as follows:
`git commit` -> `Husky` intercepts the event -> `Husky` runs `Commitlint` -> `Commitlint` validates the message -> If the message is invalid, the commit is aborted.

### Developer Setup

To activate this system on your local machine, follow these one-time setup steps:
1.  Install [Node.js](https://nodejs.org/) (version 16+ is recommended).
2.  Run the following command in the project's root directory:
    ```bash
    npm install
    ```
    This command will download all necessary development tools and automatically configure the Git hooks.

### Rules and Testing Scenarios

The commit validation rules are configured in the `.commitlintrc.json` file. The scenarios below are based on the current configuration. **If you modify the rules in `.commitlintrc.json`, remember to update these examples!**

**Current Key Rules:**
1.  **`extends: ['@commitlint/config-conventional']`**: We inherit the base ruleset from Conventional Commits (requires a type, description, etc.). This includes:
    *   A mandatory type and subject.
    *   A line length limit for the commit body (100 characters).
2.  **`header-max-length: 72`**: The header line must not exceed 72 characters.
3.  **`scope-empty: 'never'`**: The `scope` field is **mandatory** for every commit. This is a project-specific rule to ensure every change is associated with a component.
4.  **`scope-enum`**: The `scope` field must be one of the predefined values. This prevents typos and enforces consistency. The current list is: `project`, `activity`, `payment`, `activity-schema`, `payment-schema`.

---

#### ✅ Examples of Valid Commits (will pass)

| Command | Explanation |
|---|---|
| `git commit -m "feat(payment): add support for credit cards"` | A new feature (`feat`) within an allowed scope (`payment`). |
| `git commit -m "fix(project): correct typo in config"` | A bug fix (`fix`) within an allowed scope (`project`). |
| `git commit -m "refactor(activity)!: drop support for legacy API"` | A commit with a `BREAKING CHANGE` (indicated by `!`) and a valid scope. |

#### ❌ Examples of Invalid Commits (will be blocked)

| Command | Commitlint Error Message | Reason for Failure |
|---|---|---|
| `git commit -m "added login"` | `type must be one of [build, chore, ci, docs, feat, fix, ...]` | Invalid type. "added" is not a standard commit type. |
| `git commit -m "docs: update installation guide"` | `scope may not be empty` | **Missing scope.** The scope is mandatory, even for types like `docs`. |
| `git commit -m "feat(api): add new endpoint"` | `scope must be one of [project, activity, payment, ...]` | **Invalid scope.** "api" is not in the list defined in `.commitlintrc.json`. |
| `git commit -m "test: invalid scope"` | `scope may not be empty` | **Missing scope.** Commitlint correctly identifies that a scope is not provided. |
| `git commit -m "fix(payment): ... (a very long and detailed description) ..."`| `header must not be longer than 72 characters` | Header is longer than 72 characters, violating the `header-max-length` rule. |
| `git commit -m "fix:"` | `subject may not be empty` | The subject (description) after the colon is empty. |
| `git commit -m "FIX(payment): some fix"` | `type must be lower-case` | The commit type must be in lower-case. |
| `git commit -m "feat: ...a very long line without line breaks..."` | `body's lines must not be longer than 100 characters` | A line in the commit body exceeds the 100-character limit. Long lines must be wrapped. |

---

## Level 2 & 3: CI/CD Automation with GitHub Actions

While local linting ensures the quality of individual commits, the CI/CD automation handles the integration and release process. This is a two-stage process orchestrated by GitHub Actions.

### Goal

1.  **PR Title Validation:** To enforce a consistent history in the `dev` branch, especially when using "Squash and Merge", the title of every Pull Request must also follow the Conventional Commits standard.
2.  **Automated Release Preparation:** To eliminate manual version bumping and changelog generation. On every push to `dev`, the system should automatically prepare a "Release PR" that aggregates all upcoming version changes.
3.  **Automated Release Finalization:** To create official Git tags and GitHub Releases automatically when code is merged into `main`, ensuring a perfect match between the code and its documented version.

### How It Works

We use two workflows that work in tandem:

1.  **`lint-pr.yml` (The Gatekeeper):** This workflow runs on every Pull Request. It uses `commitlint` to validate the PR's title. If the title is invalid (e.g., "fix bug"), the check fails, signaling that the title needs to be corrected before merging. This is our **Level 2** protection.

2.  **`release-please.yml` (The Release Orchestrator):** This is the core of our **Level 3** automation. It contains two separate jobs triggered by pushes to different branches:
    *   **Job 1: `prepare-release-pr` (on `dev` branch):** When new features are merged into `dev`, this job runs `release-please` with the `manifest-pr` command. It analyzes the new commits, calculates the next versions for all affected components, and creates (or updates) a single, combined Pull Request named "chore: release X packages". This PR is our "Release Candidate".
    *   **Job 2: `create-release` (on `main` branch):** When code from `dev` (including the merged "Release PR") is pushed to `main`, this job runs `release-please` with the `manifest-release` command. It reads the version information from the committed files and performs the final release actions: creating Git tags (e.g., `payment-v2.1.0`) and publishing GitHub Releases with generated changelogs for each component.

### Key Configuration Files

The behavior of `release-please` is controlled by two JSON files in the root of the repository:

*   **`.release-please-config.json`:** The main configuration file. It defines all independent components (`packages`) within the monorepo, their release type (`simple`, `node`), and where their version file is located. The crucial setting `"separate-pull-requests": false` ensures that all changes are bundled into a single Release PR.
*   **`.release-please-manifest.json`:** A simple "source of truth" file that maps each component's path to its latest released version (e.g., `"services/payment": "2.0.1"`). This file is read and automatically updated by `release-please`.

---

### **CI/CD Automation: Testing Scenarios**

This section describes the expected behavior of our automated CI/CD workflows. It is divided into positive scenarios (what should happen) and negative scenarios (what should be prevented).

#### **Positive Scenarios (Expected to Succeed)**

These tests verify the main release automation flow from `dev` to `main`.

| Action | Triggering Event | Expected Result |
| :--- | :--- | :--- |
| **1. Create a "Release PR"** | Merge a `feat(activity)` commit into the `dev` branch. | A new Pull Request is automatically created by the `release-please` bot. It contains version bumps for the `activity` component in `version.txt` and `.release-please-manifest.json`, along with an updated `CHANGELOG.md`. |
| **2. Update an existing "Release PR"** | With the "Release PR" from the previous step still open, merge a `fix(payment)` commit into `dev`. | No new PR is created. Instead, the **existing** "Release PR" is updated to also include version bumps and changelog entries for the `payment` component. |
| **3. Finalize the Release** | Merge the "Release PR" into `dev`, then merge `dev` into `stg`, and finally merge `stg` into `main`. | A `push` to `main` triggers the second stage. The system creates Git tags (e.g., `activity-v1.1.0`, `payment-v2.0.2`) and publishes corresponding GitHub Releases for each component. |

#### **Negative Scenarios (Expected to be Blocked or Ignored)**

These tests ensure the system is robust and doesn't perform actions when it shouldn't.

| Action | Triggering Event | Expected Result & Reason for Failure |
| :--- | :--- | :--- |
| **1. Create a PR with an invalid title** | Open a new Pull Request into `dev` with a title like `my fixes`. | The `Lint PR Title` CI check **fails**. The reason is that the title does not follow the Conventional Commits standard, which is required for squash merges. The PR cannot be merged until the title is fixed (e.g., to `fix(payment): resolve calculation error`). |
| **2. Push a non-release commit to `dev`** | Merge a commit like `docs(readme): update instructions` or `chore: refactor tooling` into `dev`. | The `Release Please` workflow runs, but **does nothing**. No "Release PR" is created or updated because `docs`, `chore`, `style`, `test`, and `refactor` are not considered release-triggering commit types by default. |

---