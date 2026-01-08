# Synology GitLab AI Review Bot Setup

This is a fork of [Schickli/agent-for-gitlab](https://github.com/Schickli/agent-for-gitlab) with **auto-review on MR open** support.

## Features

| Feature                    | Trigger                                             |
| -------------------------- | --------------------------------------------------- |
| Auto-review on MR open     | Automatic when MR is created                        |
| On-demand via @ai          | Comment `@ai <request>` on MR/Issue                 |
| Project-specific knowledge | deepsearch (Vue/TS), OrangeDrive (C++), conan-index |

## Quick Start

### 1. Create GitLab Bot User

```bash
# At https://git.synology.inc/admin/users/new
Username: ai-reviewer
Name: AI Code Reviewer
Email: ai-reviewer@synology.com
```

### 2. Create Bot Access Token

1. Login as `ai-reviewer`
2. Go to: User Settings > Access Tokens
3. Create token with scopes: `api`, `read_repository`, `write_repository`
4. Save the token

### 3. Add Bot to Group

1. Go to: Group > Settings > Members
2. Invite `ai-reviewer` with **Developer** role

### 4. Deploy Webhook Service

```bash
# On your internal VM
cd /opt
git clone https://github.com/solarispika/agent-for-gitlab.git
cd agent-for-gitlab/gitlab-app

# Create .env
cat > .env << 'EOF'
GITLAB_URL=https://git.synology.inc
GITLAB_TOKEN=<your-bot-token>
WEBHOOK_SECRET=<generate-with-openssl-rand-hex-32>
PORT=3000

AI_GITLAB_USERNAME=ai-reviewer
AI_GITLAB_EMAIL=ai-reviewer@synology.com

TRIGGER_PHRASE=@ai
OPENCODE_MODEL=anthropic/claude-sonnet-4

AUTO_REVIEW_ENABLED=true
RATE_LIMITING_ENABLED=false
CANCEL_OLD_PIPELINES=true
BRANCH_PREFIX=ai

LOG_LEVEL=info
LOG_FORMAT=json
EOF

# Generate webhook secret
openssl rand -hex 32

# Start without Redis (rate limiting disabled)
docker-compose -f docker-compose.no-redis.yml up -d

# Check logs
docker-compose -f docker-compose.no-redis.yml logs -f
```

### 5. Configure GitLab Group Webhook

1. Go to: `https://git.synology.inc/YOUR_GROUP/-/hooks`
2. Add webhook:
   - **URL**: `http://<your-vm-ip>:3000/webhook`
   - **Secret token**: Same as `WEBHOOK_SECRET` in `.env`
   - **Triggers**:
     - [x] Comments
     - [x] Merge request events
3. Save and test

### 6. Add CI Configuration to Repos

Add to each repo's `.gitlab-ci.yml`:

```yaml
include:
  - project: "your-group/agent-for-gitlab"
    file: "/gitlab-utils/ai-review.gitlab-ci.yml"
```

Or copy the content from `gitlab-utils/ai-review.gitlab-ci.yml`.

### 7. Add GitLab CI/CD Variables (Group Level)

Go to: Group > Settings > CI/CD > Variables

| Variable         | Value                                                    | Protected | Masked |
| ---------------- | -------------------------------------------------------- | --------- | ------ |
| `GITLAB_TOKEN`   | Bot's access token                                       | Yes       | Yes    |
| `AI_AGENT_IMAGE` | `ghcr.io/schickli/ai-code-for-gitlab/agent-image:latest` | No        | No     |

## Testing

### Test 1: Health Check

```bash
curl http://<your-vm-ip>:3000/health
# Expected: ok
```

### Test 2: Auto-Review

1. Create a new MR in any repo
2. Wait ~30 seconds
3. Check MR for review comment

### Test 3: On-Demand

Comment on any MR:

```
@ai What does this code change?
```

## Configuration Reference

| Variable                | Description                        | Default                     |
| ----------------------- | ---------------------------------- | --------------------------- |
| `GITLAB_URL`            | GitLab instance URL                | `https://gitlab.com`        |
| `GITLAB_TOKEN`          | Bot's access token                 | Required                    |
| `WEBHOOK_SECRET`        | Webhook verification secret        | Required                    |
| `AUTO_REVIEW_ENABLED`   | Enable auto-review on MR open      | `true`                      |
| `TRIGGER_PHRASE`        | Trigger phrase for on-demand       | `@ai`                       |
| `OPENCODE_MODEL`        | LLM model                          | `anthropic/claude-sonnet-4` |
| `RATE_LIMITING_ENABLED` | Enable rate limiting (needs Redis) | `true`                      |
| `CANCEL_OLD_PIPELINES`  | Cancel old pending pipelines       | `true`                      |
