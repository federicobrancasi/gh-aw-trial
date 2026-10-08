---
name: Changelog Video
description: >-
    Makes a short explainer video (1-2 minutes, narrated, captioned) of one microsoft/vscode pull request with
    pr-studio (https://github.com/federicobrancasi/pr-studio). The setup steps download the PR and the files it
    changes; the agent then makes the video with no network access. The video is the run's artifact.

on:
    workflow_dispatch:
        inputs:
            pr_number:
                description: 'microsoft/vscode pull request number'
                required: true
                type: string

run-name: 'Changelog video -- microsoft/vscode#${{ inputs.pr_number }}'

concurrency:
    job-discriminator: ${{ inputs.pr_number }}

timeout-minutes: 45

permissions:
    contents: read
    copilot-requests: write

engine:
    id: copilot
    model: claude-sonnet-5.5
    args: ["--reasoning-effort", "high"]
    env:
        # pr-studio's offline mode: GitHub data only from what the setup downloaded, the local voice.
        PRS_OFFLINE: '1'
        PRS_DELIVER: out

max-ai-credits: 600

sandbox:
    agent:
        memory: 6g

# pr-studio is the agent's working folder: its instructions, CLI, diagram catalog and reviewer agent.
checkout:
    - repository: federicobrancasi/pr-studio
      ref: d5ab39a0085acdde4beaa1102e370bf6a126e624
      path: .
      current: true

# No network: everything the agent needs is downloaded by the setup steps below.
network: {}

# The video is the run's artifact: no writes to GitHub, and no issue when a run fails. The agent records the
# published video once (record-video), which also replaces gh-aw's default create-issue output.
safe-outputs:
    missing-data: false
    missing-tool: false
    report-failure-as-issue: false
    report-incomplete: false
    report-failed-jobs: false
    jobs:
        record-video:
            description: Record the published video, once, when it is done.
            runs-on: ubuntu-latest
            permissions: {}
            output: Video recorded.
            inputs:
                headline:
                    description: The headline of the published video.
                    required: true
                    type: string
                review:
                    description: The pr-director verdict on the published video.
                    required: true
                    type: choice
                    options: [signed-off, changes-requested, not-reviewed]
            steps:
                - name: Verify one video record
                  shell: bash
                  run: |
                      set -euo pipefail
                      test "$(jq '[.items[] | select(.type == "record_video")] | length' "$GH_AW_AGENT_OUTPUT")" = "1"
                      jq -r '.items[] | select(.type == "record_video") | "Video: \(.headline) (review: \(.review))"' "$GH_AW_AGENT_OUTPUT"

tools:
    github: false
    edit:
    # Rendering, encoding and the sync check can each take a few minutes.
    timeout: 900
    bash:
        - "cd *"
        - "node *"
        - "python3 *"
        - "cat *"
        - "ls *"
        - "grep *"
        - "head *"
        - "tail *"
        - "sed *"
        - "awk *"
        - "cut *"
        - "tr *"
        - "wc *"
        - "sort *"
        - "uniq *"
        - "find *"
        - "echo *"
        - "printf *"
        - "jq *"
        - "mkdir *"
        - "cp *"
        - "mv *"
        - "rm *"
        - "sleep *"
        - "ps *"
        - "date *"
        - "test *"
        - "diff *"
        - "ffprobe *"
        - "git status *"
        - "git diff *"
        - "git log *"

pre-agent-steps:
    # pr-studio's own setup: node packages, Chrome, the voice environment (Kokoro, Whisper, ffmpeg), cached.
    - name: Set up pr-studio
      uses: ./.github/actions/setup
    - name: Download the pull request and the files it changes
      shell: bash
      env:
          PR_NUMBER: ${{ inputs.pr_number }}
          GITHUB_TOKEN: ${{ github.token }}
      run: |
          set -euo pipefail
          if ! [[ "$PR_NUMBER" =~ ^[1-9][0-9]{0,9}$ ]]; then
              echo "::error::Invalid PR number: $PR_NUMBER"
              exit 1
          fi
          node cli/prs.mjs new "microsoft/vscode#$PR_NUMBER"

post-steps:
    - name: Upload the video
      if: always()
      uses: actions/upload-artifact@v7.0.1
      with:
          name: video-${{ inputs.pr_number }}
          path: |
              out/
              videos/*/notes.md
          if-no-files-found: warn
          retention-days: 14
---

# Changelog video

Make the explainer video of https://github.com/microsoft/vscode/pull/${{ inputs.pr_number }}.

Nobody is watching this run: follow `.claude/skills/pr-video/guides/unattended.md` and the `pr-video` skill it points
to. There is no network: the setup already downloaded the pull request and the files it changes (`PRS_OFFLINE` is
set, and `node cli/prs.mjs new` continues from that download).

When the video is published, record it once with the `record-video` tool: its headline and the reviewer's verdict.
