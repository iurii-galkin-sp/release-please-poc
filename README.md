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
    *   **A line length limit for the commit body (100 characters).**
2.  **`header-max-length: 72`**: The header line must not exceed 72 characters.
3.  **`scope-enum`**: The `scope` field must be one of the predefined values. This prevents typos and enforces consistency. The current list is: `project`, `activity`, `payment`, `activity-schema`, `payment-schema`.

---

#### ✅ Examples of Valid Commits (will pass)

| Command                                                              | Explanation                                                                     |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `git commit -m "feat(payment): add support for credit cards"`        | A new feature (`feat`) within an allowed scope (`payment`).                     |
| `git commit -m "fix(project): correct typo in config"`               | A bug fix (`fix`) within an allowed scope (`project`).                          |
| `git commit -m "docs: update installation guide"`                    | A commit without a `scope` (which is allowed by the standard).                  |
| `git commit -m "refactor!: drop support for legacy API"`             | A commit with a `BREAKING CHANGE` (indicated by `!`).                           |

#### ❌ Examples of Invalid Commits (will be blocked)

| Command                                                                | Reason for Failure                                                              | Commitlint Error Message                                                              |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `git commit -m "added login"`                                          | Invalid type. "added" is not a standard commit type.                            | `✖ type must be one of [build, chore, ci, docs, feat, fix, ...]`                      |
| `git commit -m "feat(api): add new endpoint"`                          | Invalid `scope`. "api" is not in the list defined in `.commitlintrc.json`.      | `✖ scope must be one of [project, activity, payment, ...]`                            |
| `git commit -m "fix(payment): ... (a very long and detailed description) ..."` | Header is longer than 72 characters, violating the `header-max-length` rule.    | `✖ header must not be longer than 72 characters`                                      |
| `git commit -m "fix:"`                                                 | The subject (description) after the colon is empty.                             | `✖ subject may not be empty`                                                          |
| `git commit -m "FIX(payment): some fix"`                               | The commit type must be in lower-case.                                          | `✖ type must be lower-case`                                                           |
| `git commit -m "feat: ...a very long line without line breaks..."`     | A line in the commit body exceeds the 100-character limit. Long lines must be wrapped. | `✖ body's lines must not be longer than 100 characters`        |

---

## Level 4: Branch Protection Rules

To enforce our `main <- stg <- dev` workflow and prevent common errors, a set of protection rules is configured for each primary branch. These rules are the technical foundation of our development process.

### Goal

1.  **Prevent Direct Pushes:** To ensure all code changes go through a formal review process, direct pushes to `main`, `stg`, and `dev` are prohibited.
2.  **Enforce Pull Requests:** All changes must be submitted via a Pull Request.
3.  **Mandate Quality Checks:** To maintain code quality and stability, all required status checks (like tests, builds, and linters) must pass before a PR can be merged.
4.  **Enforce Merge Strategy:** To maintain a clean and readable Git history, a specific merge strategy is enforced depending on the target branch (`Squash` for `dev`, `Merge Commit` for `main`/`stg`).

### How It Works

We have three sets of branch protection rules configured in the repository settings (`Settings -> Branches`).

*   **For the `main` and `stg` branches:** The rules are strict. They require a PR, all status checks must pass, and the branch must be up-to-date with its target before merging. This ensures maximum stability.
*   **For the `dev` branch:** The rules are similar but optimized for a faster development pace. Specifically, it does not require the branch to be up-to-date before merging, as `dev` is a very active branch.
*   **Merge Strategy Enforcement:** The choice of merge method is a **process-level agreement**, enforced during code review. In the repository settings (`Settings -> General -> Pull Requests`), only "Allow merge commits" and "Allow squash merging" are enabled, while "Allow rebase merging" is disabled entirely.

This section provides the exact configuration for the branch protection rules required by our workflow. These rules are configured under **`Settings > Branches > Branch protection rules`**.

### **Rule 1: `main` Branch Protection**

**Objective:** Maximum stability. This is our production branch.

1.  **Branch name pattern**: `main`
2.  **Enable** ✅ `Require a pull request before merging`.
    *   `Require approvals`: Set to `0` (for now, as a solo developer).
3.  **Enable** ✅ `Dismiss stale pull request approvals when new commits are pushed`.
4.  **Enable** ✅ `Require status checks to pass before merging`.
    *   **Enable** ✅ `Require branches to be up to date before merging`. **CRITICAL for `main`!**
    *   **Add checks**:
        *   `Check PR Title`
        *   *(Future)* `build`
        *   *(Future)* `test`
        *   *(Future)* `Gatekeeper: main`
5.  **Enable** ✅ `Do not allow bypassing the above settings`.
6.  **Under `Rules applied to everyone including administrators`:**
    *   **Disable** ⬜ `Allow force pushes`.
    *   **Disable** ⬜ `Allow deletions`.

---

### **Rule 2: `stg` Branch Protection**

**Objective:** Pre-production stability. Mirrors `main` rules.

1.  **Branch name pattern**: `stg`
2.  **Enable** ✅ `Require a pull request before merging`.
    *   `Require approvals`: Set to `0`.
3.  **Enable** ✅ `Dismiss stale pull request approvals...`.
4.  **Enable** ✅ `Require status checks to pass before merging`.
    *   **Enable** ✅ `Require branches to be up to date before merging`.
    *   **Add checks**: Same as `main`.
5.  **Enable** ✅ `Do not allow bypassing the above settings`.
6.  **Under `Rules applied to everyone including administrators`:**
    *   **Disable** ⬜ `Allow force pushes`.
    *   **Disable** ⬜ `Allow deletions`.

---

### **Rule 3: `dev` Branch Protection**

**Objective:** Ensure a clean, conventional commit history for `release-please`.

1.  **Branch name pattern**: `dev`
2.  **Enable** ✅ `Require a pull request before merging`.
    *   `Require approvals`: Set to `0`.
3.  **Enable** ✅ `Dismiss stale pull request approvals...`.
4.  **Enable** ✅ `Require status checks to pass before merging`.
    *   **Disable** ⬜ `Require branches to be up to date before merging`. **IMPORTANT!** This is disabled to speed up the development workflow on this active branch.
    *   **Add checks**:
        *   `Check PR Title`
        *   *(Future)* `build`
        *   *(Future)* `test`
        *   *(Future)* `Gatekeeper: dev`
5.  **Enable** ✅ `Do not allow bypassing the above settings`.
6.  **Under `Rules applied to everyone including administrators`:**
    *   **Disable** ⬜ `Allow force pushes`.
    *   **Disable** ⬜ `Allow deletions`.

---

### **Repository-Wide Merge Strategy Configuration**

This is configured under **`Settings > General > Pull Requests`**.

1.  **Disable** ⬜ `Allow rebase merging`. This method rewrites history and is forbidden.
2.  **Enable** ✅ `Allow squash merging`. **This method MUST be used when merging into `dev`**.
3.  **Enable** ✅ `Allow merge commits`. **This method MUST be used when merging into `stg` and `main`**.

---

### Testing Scenarios for Branch Protection Rules

This section describes how to verify that the branch protection rules are working as expected.

#### **Positive Scenarios (Expected to Succeed)**

| Action to Test | How to Verify | Expected Result |
| :--- | :--- | :--- |
| **Correct Merge into `dev`** | Open a PR into `dev` and click the "Merge" button dropdown. | The **"Squash and merge"** option is available and should be selected. The PR merges successfully after all checks pass. |
| **Correct Merge into `main`** | Open a PR from `stg` into `main` and click the "Merge" button dropdown. | The **"Create a merge commit"** option is available and should be selected. The PR merges successfully after all checks pass. |

#### **Negative Scenarios (Expected to be Blocked)**

| Action to Test | How to Verify | Expected Result & Reason for Failure |
| :--- | :--- | :--- |
| **Direct Push is Blocked** | From your local machine, try to push a commit directly to `dev` using `git push origin dev`. | The push is **rejected** by Git. The remote server will return an error like `(pre-receive hook declined)` or `(protected branch hook declined)`, because direct pushes are prohibited. |
| **Merge is Blocked if Checks Fail** | Open a PR with a failing status check (e.g., an invalid title that fails the `Check PR Title` check). | The "Merge pull request" button is **disabled (greyed out)**. A message indicates that required status checks have not passed. |
| **Rebase Merging is Not Possible** | Open any PR and click the "Merge" button dropdown. | The **"Rebase and merge"** option is completely **missing** from the list, as it has been disabled in the repository settings. |
| **Merge is Blocked if `main` is not Up-to-Date** | Open a PR targeting `main`. While it's open, merge another PR into `main`. Then, return to the first PR. | The PR is now marked as "out-of-date". The "Merge" button is **disabled** until you click "Update branch", because the `main` branch requires branches to be up-to-date before merging. |

### **Testing and Verification Plan**

| Test Case | How to Verify | Expected Result & Reason |
| :--- | :--- | :--- |
| **TC-1: Block direct push to `dev`** | `git checkout dev && git commit --allow-empty -m "test" && git push` | **Push REJECTED**. Reason: Branch is protected, requiring a pull request. |
| **TC-2: Block merge on failed check** | Open a PR to `dev` with an invalid title (e.g., "my pr"). | **Merge button DISABLED**. Reason: `Check PR Title` status check is failing. |
| **TC-3: Block merge if `main` is outdated** | Open a PR to `main`. While open, merge another PR into `main`. | **Merge button DISABLED** on the first PR. Reason: Branch must be updated before merging. |
| **TC-4: Verify correct merge into `dev`** | Open a PR to `dev`. All checks pass. | **"Squash and merge"** option is available and should be used. |
| **TC-5: Verify correct merge into `main`**| Open a PR from `stg` to `main`. All checks pass. | **"Create a merge commit"** option is available and should be used. |

### **Rule 4: `stg` Branch Ruleset**

**Objective:** Ensure pre-production stability. This branch must be as stable as `main`.

**Configuration Path:** `Settings > Branches > Rulesets > New ruleset`

1.  **`Ruleset name`**: `Staging Branch Rules`
2.  **`Enforcement status`**: `Active`
3.  **`Target branches`**:
    *   `Add a target`: `Include refs (branches)`
    *   `Matching pattern`: `stg`
4.  **`Branch protections`**:
    *   ✅ **Enable** `Restrict deletions`.
    *   ✅ **Enable** `Require a pull request before merging`.
        *   `Required approvals`: Set to `0`.
        *   ✅ **Enable** `Dismiss stale approvals...`.
    *   ✅ **Enable** `Require status checks to pass before merging`.
        *   ✅ **Enable** `Require branches to be up to date before merging`. **CRITICAL for `stg`!**
        *   **Add checks**: `Check PR Title`, `build`, `test`, `Gatekeeper: stg` (future).
    *   ✅ **Enable** `Restrict force pushes`.
5.  **`Merge controls`**:
    *   ✅ **Block "Squash merge"**. REASON: We must preserve the clean commit history coming from `dev`.
    *   ✅ **Block "Rebase merge"**. REASON: Rewriting history is forbidden.
    *   ⬜ **Do not block "Merge commit"**. This is the only allowed method.

---

### **Rule 5: `dev` Branch Ruleset**

**Objective:** Enforce a clean, feature-based commit history and a fast development pace.

**Configuration Path:** `Settings > Branches > Rulesets > New ruleset`

1.  **`Ruleset name`**: `Development Branch Rules`
2.  **`Enforcement status`**: `Active`
3.  **`Target branches`**:
    *   `Add a target`: `Include refs (branches)`
    *   `Matching pattern`: `dev`
4.  **`Branch protections`**:
    *   ✅ **Enable** `Restrict deletions`.
    *   ✅ **Enable** `Require a pull request before merging`.
        *   `Required approvals`: Set to `0`.
        *   ✅ **Enable** `Dismiss stale approvals...`.
    *   ✅ **Enable** `Require status checks to pass before merging`.
        *   ⬜ **Disable** `Require branches to be up to date before merging`. **IMPORTANT!** This is disabled to increase workflow velocity on this active branch.
        *   **Add checks**: `Check PR Title`, `build`, `test`, `Gatekeeper: dev` (future).
    *   ✅ **Enable** `Restrict force pushes`.
5.  **`Merge controls`**:
    *   ✅ **Block "Merge commit"**. REASON: We must enforce a clean, linear history. Each PR should become one commit.
    *   ✅ **Block "Rebase merge"**. REASON: Rewriting history is forbidden.
    *   ⬜ **Do not block "Squash merge"**. This is the only allowed method.

---

### **Testing and Verification Plan for `stg` and `dev` Rulesets**

| Test Case | How to Verify | Expected Result & Reason |
| :--- | :--- | :--- |
| **TC-6: Enforce Squash on `dev`** | Open a PR into `dev`. After all checks pass, click the "Merge" button. | The merge dialog **only shows the "Squash and merge"** option. Other methods are blocked and unavailable. |
| **TC-7: Enforce Merge Commit on `stg`** | Open a PR from `dev` into `stg`. After all checks pass, click the "Merge" button. | The merge dialog **only shows the "Create a merge commit"** option. Other methods are blocked. |
| **TC-8: Outdated Branch Block on `stg`** | Open a PR to `stg`. While it's open, push another commit directly to `stg` (if possible, or merge another PR). | The first PR is now **blocked from merging**. The "Update branch" button appears. Reason: `stg` requires branches to be up-to-date. |
| **TC-9: Outdated Branch Ignored on `dev`**| Open a PR to `dev`. While it's open, merge another PR into `dev`. | The first PR is **NOT blocked from merging** (it may show a warning, but the button remains active). Reason: `dev` does not require branches to be up-to-date. |

---

## Level 5: Automated Release System (Release Please)

To automate versioning and release notes generation, this project uses Google's `release-please` tool. The system is built around a two-stage workflow that separates release preparation from finalization.

### Goal

1.  **Automate Version Bumping:** Automatically determine the next semantic version for each component based on Conventional Commits.
2.  **Generate Changelogs:** Create and maintain `CHANGELOG.md` files for each component.
3.  **Streamline Releases:** Automate the creation of Git tags and GitHub Releases, ensuring a consistent and auditable release history.

### How It Works

The process is divided into two automated stages, managed by separate GitHub Actions workflows:

1.  **Preparation Stage (on `dev` branch):**
    *   **Trigger:** A push to the `dev` branch.
    *   **Action:** The `release-please-prepare.yml` workflow runs `release-please` to analyze new commits.
    *   **Result:** It creates or updates a single "Release PR" targeting `dev`. This PR contains only version bumps in `version.txt` files and updated `CHANGELOG.md` files for all changed components. This allows the team to review upcoming releases before they are locked in.

2.  **Finalization Stage (on `main` branch):**
    *   **Trigger:** A push to the `main` branch (which happens after a `dev -> stg -> main` merge).
    *   **Action:** The `release-please-finalize.yml` workflow runs. It looks for commits from a merged Release PR.
    *   **Result:** If found, it creates component-specific Git tags (e.g., `payment-v2.0.2`) and publishes corresponding GitHub Releases with auto-generated release notes.

### Configuration Files

The entire system is configured via the following files in the repository root:

*   **`.release-please-config.json`**: The main configuration file.
    *   `"separate-pull-requests": false` - This is set to `false` for the PoC to consolidate all component updates into a single, easy-to-review Release PR.
    *   `"bump-minor-pre-major": true` - Ensures that for pre-1.0.0 versions, a `feat` commit correctly bumps the minor version (e.g., `0.1.0` -> `0.2.0`), following SemVer conventions for unstable packages.
    *   `"packages"` - This object defines all trackable components within the monorepo. The key is the path to the component, and the value contains its specific configuration.
        *   `"release-type": "simple"`: A generic release type that updates a `version.txt` file. It's used for all components as they are not standard npm packages.
        *   `"component"`: A custom name used for creating clean Git tags (e.g., `payment-v1.0.0` instead of `services/payment-v1.0.0`).

*   **`.release-please-manifest.json`**: This file acts as a database for `release-please`, storing the last released version for each component. The tool uses this file to determine which commits are new. It is automatically updated by `release-please`.

*   **`.github/workflows/`**: This directory contains the GitHub Actions that drive the process.
    *   `lint-pr-title.yml`: Ensures PR titles follow the Conventional Commits standard, which is crucial for the "squash merge" strategy.
    *   `release-please-prepare.yml`: Manages the release preparation stage on the `dev` branch.
    *   `release-please-finalize.yml`: Manages the release finalization stage on the `main` branch.
    