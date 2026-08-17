# Multi-Forge Mirroring Cheatsheet (GitHub → GitLab & Codeberg)

This setup uses **Jujutsu (`jj`)** locally and a single **reusable GitHub Action** (`nexnc/workflows`) to automatically create repositories on GitLab & Codeberg, sync descriptions/topics, and mirror all branches and tags.

---

## 1. Prerequisite: Central Reusable Workflow

Host this file in a **public** repository named `nexnc/workflows` at path `.github/workflows/mirror.yml`:

```yaml
name: Reusable Multi-Forge Mirror

on:
  workflow_call:
    inputs:
      repo_name:
        type: string
        required: false
        default: ""
        description: "Override repository name if different from GitHub"

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Auto-create, sync metadata & push to GitLab
        env:
          DESCRIPTION: ${{ github.event.repository.description || '' }}
          TOPICS: ${{ toJson(github.event.repository.topics) }}
        run: |
          TARGET="${{ inputs.repo_name != '' && inputs.repo_name || github.event.repository.name }}"

          # 1. Create project if missing
          curl --silent --request POST "[https://gitlab.com/api/v4/projects](https://gitlab.com/api/v4/projects)" \
            --header "PRIVATE-TOKEN: ${{ secrets.GITLAB_TOKEN }}" \
            --header "Content-Type: application/json" \
            --data "{\"name\": \"${TARGET}\", \"visibility\": \"public\"}" > /dev/null || true

          # 2. Sync Description and Topics
          GITLAB_TAGS=$(echo "$TOPICS" | jq -r 'join(",")')
          curl --silent --request PUT "[https://gitlab.com/api/v4/projects/nexnc%2F$](https://gitlab.com/api/v4/projects/nexnc%2F$){TARGET}" \
            --header "PRIVATE-TOKEN: ${{ secrets.GITLAB_TOKEN }}" \
            --header "Content-Type: application/json" \
            --data "{\"description\": \"${DESCRIPTION}\", \"tag_list\": \"${GITLAB_TAGS}\"}" > /dev/null || true

          # 3. Push all branches and tags
          git push --prune --force "https://oauth2:${{ secrets.GITLAB_TOKEN }}@[gitlab.com/nexnc/$](https://gitlab.com/nexnc/$){TARGET}.git" \
            "+refs/remotes/origin/*:refs/heads/*" \
            "+refs/tags/*:refs/tags/*"

      - name: Auto-create, sync metadata & push to Codeberg
        env:
          DESCRIPTION: ${{ github.event.repository.description || '' }}
          TOPICS: ${{ toJson(github.event.repository.topics) }}
        run: |
          TARGET="${{ inputs.repo_name != '' && inputs.repo_name || github.event.repository.name }}"

          # 1. Create repo if missing
          curl --silent --request POST "[https://codeberg.org/api/v1/user/repos](https://codeberg.org/api/v1/user/repos)" \
            --header "Authorization: token ${{ secrets.CODEBERG_TOKEN }}" \
            --header "Content-Type: application/json" \
            --data "{\"name\": \"${TARGET}\", \"private\": false, \"auto_init\": false}" > /dev/null || true

          # 2. Sync Description
          curl --silent --request PATCH "[https://codeberg.org/api/v1/repos/nexnc/$](https://codeberg.org/api/v1/repos/nexnc/$){TARGET}" \
            --header "Authorization: token ${{ secrets.CODEBERG_TOKEN }}" \
            --header "Content-Type: application/json" \
            --data "{\"description\": \"${DESCRIPTION}\"}" > /dev/null || true

          # 3. Sync Topics
          curl --silent --request PUT "[https://codeberg.org/api/v1/repos/nexnc/$](https://codeberg.org/api/v1/repos/nexnc/$){TARGET}/topics" \
            --header "Authorization: token ${{ secrets.CODEBERG_TOKEN }}" \
            --header "Content-Type: application/json" \
            --data "{\"topics\": ${TOPICS}}" > /dev/null || true

          # 4. Push all branches and tags
          git push --prune --force "https://${{ secrets.CODEBERG_TOKEN }}@codeberg.org/nexnc/${TARGET}.git" \
            "+refs/remotes/origin/*:refs/heads/*" \
            "+refs/tags/*:refs/tags/*"
```

---

## 2. Token Requirements Reference

| Platform | Location | Required Permissions / Scopes | Secret Name |
| :--- | :--- | :--- | :--- |
| **GitLab** | `User Settings` → `Access Tokens` | `api`, `write_repository` | `GITLAB_TOKEN` |
| **Codeberg** | `Settings` → `Applications` | `repository: Read and write`<br>`user: Read and write` | `CODEBERG_TOKEN` |

---

## 3. Standard Routine for Every New Project

### Step 1: Create GitHub Repo & Set Secrets
* Create a new repository: `https://github.com/new` (e.g., `nexnc/<project-name>`).
* Navigate to **Settings** → **Secrets and variables** → **Actions**.
* Add the following repository secrets:
  * `GITLAB_TOKEN`
  * `CODEBERG_TOKEN`

### Step 2: Initialize Locally with `jj`
```bash
mkdir <project-name> && cd <project-name>
jj git init --colocated
jj git remote add origin git@github.com:nexnc/<project-name>.git
```

### Step 3: Add the Caller Workflow
Create `.github/workflows/mirror.yml` in your project root:

```yaml
name: Mirror

on:
  push:
    branches: ["*"]
    tags: ["*"]

jobs:
  sync:
    uses: nexnc/workflows/.github/workflows/mirror.yml@main
    secrets: inherit
```

### Step 4: Commit and Push
```bash
jj describe -m "feat: initial commit with multi-forge mirror"
jj bookmark create main -r @
jj git push --allow-new --bookmark main
```

---

## 4. What Happens on Every Push

* **Auto-creation:** GitLab and Codeberg repositories are created via API if they do not exist.
* **Metadata Sync:** Repository description and topic tags from GitHub are synced to both platforms.
* **Code Mirror:** All branches (`refs/heads/*`) and tags (`refs/tags/*`) are force-pushed across both remotes.
