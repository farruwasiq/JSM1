- name: Authenticate GitHub CLI
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          echo "Authenticating GitHub CLI..."
          gh auth setup-git

      - name: Find Latest Successful 'start' Run
        id: find_run
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          set -eo pipefail # Exit immediately if any command in the pipe fails

          # Get the workflow ID for "JSM MW"
          WORKFLOW_ID=$(gh api repos/${{ github.repository }}/actions/workflows --jq '.workflows[] | select(.name=="JSM MW") | .id')

          if [ -z "$WORKFLOW_ID" ]; then
            echo "::error::Workflow 'JSM MW' not found. Exiting."
            exit 1
          fi

          # Fetch workflow runs for this ID, extracting individual objects for local jq filtering
          LATEST_START_RUN_ID=$(gh api /repos/${{ github.repository }}/actions/workflows/$WORKFLOW_ID/runs \
            --jq '.workflow_runs[]' --paginate | \
            jq -r 'select(
              (.display_title | contains("JSM MW - start")) and
              (.event == "workflow_dispatch") and
              (.status == "completed") and
              (.conclusion == "success")
            ) | .id | tostring' | head -n 1)

          if [ -z "$LATEST_START_RUN_ID" ]; then
            echo "::error::No successful 'start' runs found with the expected criteria (displayTitle: 'JSM MW - start', event: 'workflow_dispatch', status: 'completed', conclusion: 'success')."
            echo "Please ensure a 'start' workflow has run successfully with these characteristics, and its run name includes 'JSM MW - start'."
            exit 1
          fi

          echo "Found latest successful 'start' run ID: $LATEST_START_RUN_ID"
          echo "run_id_to_download=$LATEST_START_RUN_ID" >> "$GITHUB_OUTPUT"