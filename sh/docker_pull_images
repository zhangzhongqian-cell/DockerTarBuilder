#!/bin/bash

# === Configuration ===
GITHUB_TOKEN=${GITHUB_TOKEN:-""}
REPO="terryWJA/DockerTarBuilder"
WORKFLOW_FILE="amd64-to_acr.yml"
BRANCH="master"          # 注意：你这里写的是 master，但仓库默认是 main？请确认！
ACR_REGISTRY="registry.cn-beijing.aliyuncs.com/docker_io_remote"
MAX_WAIT_MINUTES=10

# === Step 1: Prompt user for Docker image ===
read -p "Enter Docker image to build and push to ACR (e.g. hello-world or nginx:latest): " IMAGE_INPUT
if [[ -z "$IMAGE_INPUT" ]]; then
  echo "❌ Error: Image name cannot be empty!"
  exit 1
fi

# === Step 2: Trigger GitHub Actions workflow ===
echo "🚀 Triggering GitHub Actions workflow..."
RESPONSE=$(curl -s -w "%{http_code}" -L \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  "https://api.github.com/repos/$REPO/actions/workflows/$WORKFLOW_FILE/dispatches" \
  -d "{\"ref\":\"$BRANCH\",\"inputs\":{\"docker_images\":\"$IMAGE_INPUT\"}}")

HTTP_CODE="${RESPONSE: -3}"
if [[ "$HTTP_CODE" != "204" ]]; then
  echo "❌ Failed to trigger workflow! HTTP status: $HTTP_CODE"
  echo "Response: ${RESPONSE%???}"
  exit 1
fi
echo "✅ Workflow triggered successfully!"

# === Step 3: Poll for workflow completion ===
echo "⏳ Waiting for workflow to complete (max ${MAX_WAIT_MINUTES} minutes)..."
POLL_INTERVAL=15
MAX_ATTEMPTS=$((MAX_WAIT_MINUTES * 60 / POLL_INTERVAL))

for ((i=1; i<=MAX_ATTEMPTS; i++)); do
  sleep $POLL_INTERVAL

  RUN_INFO=$(curl -s \
    -H "Authorization: Bearer $GITHUB_TOKEN" \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    "https://api.github.com/repos/$REPO/actions/workflows/$WORKFLOW_FILE/runs?branch=$BRANCH&event=workflow_dispatch")

  STATUS=$(echo "$RUN_INFO" | jq -r '.workflow_runs[0].status // empty')
  CONCLUSION=$(echo "$RUN_INFO" | jq -r '.workflow_runs[0].conclusion // empty')
  RUN_ID=$(echo "$RUN_INFO" | jq -r '.workflow_runs[0].id // empty')

  if [[ -z "$STATUS" || "$STATUS" == "null" ]]; then
    echo "⚠️  No recent run found, continuing to wait..."
    continue
  fi

  echo "🔍 Current status: $STATUS (conclusion: $CONCLUSION)"

  if [[ "$STATUS" == "completed" ]]; then
    if [[ "$CONCLUSION" == "success" ]]; then
      echo "✅ Workflow completed successfully!"
      break
    else
      echo "❌ Workflow failed! Conclusion: $CONCLUSION"
      echo "View logs at: https://github.com/$REPO/actions/runs/$RUN_ID"
      exit 1
    fi
  fi

  if [[ $i -eq $MAX_ATTEMPTS ]]; then
    echo "⏰ Timeout! Workflow did not complete in time."
    echo "Check manually: https://github.com/$REPO/actions"
    exit 1
  fi
done

# === Step 4: Pull from ACR with flattened name ===
FULL_IMAGE="$IMAGE_INPUT"
if [[ "$FULL_IMAGE" != *":"* ]]; then
  FULL_IMAGE="${FULL_IMAGE}:latest"
fi

# Flatten image name: remove any leading namespace (e.g., nginx/xxx -> xxx)
ACR_FLATTENED_NAME=$(echo "$FULL_IMAGE" | sed 's|.*/||')
ACR_IMAGE="$ACR_REGISTRY/$ACR_FLATTENED_NAME"
ORIGINAL_IMAGE="$FULL_IMAGE"  # preserve original user input for tagging

echo "📥 Pulling from ACR: $ACR_IMAGE"
docker pull "$ACR_IMAGE"

if [ $? -ne 0 ]; then
  echo "❌ Failed to pull image from ACR!"
  echo "Possible reasons:"
  echo "1. ACR namespace 'docker_io_remote' is private — log in first:"
  echo "   docker login registry.cn-beijing.aliyuncs.com"
  echo "2. The image hasn't been pushed yet — check GitHub Actions status."
  echo "3. Make sure your workflow flattens image names (e.g., 'nginx/xxx' → 'xxx')."
  exit 1
fi

# === Step 5: Tag back to original name ===
echo "🏷️  Tagging as original name: $ORIGINAL_IMAGE"
docker tag "$ACR_IMAGE" "$ORIGINAL_IMAGE"

if [ $? -eq 0 ]; then
  echo "✅ Successfully tagged $ACR_IMAGE as $ORIGINAL_IMAGE"
else
  echo "❌ Failed to tag image!"
  exit 1
fi

# Show result
echo "📋 Local image:"
docker images "$ORIGINAL_IMAGE"