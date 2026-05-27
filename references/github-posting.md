# GitHub Posting (Step 13)

Only invoked after the user reviews the proposed comments and confirms which to post.
Never post automatically.

## Comment body format

No severity labels, no `[ID]` prefixes. Plain concern format:

```
**<one-line title>**

**What's the concern?**
- <bullet>
- <bullet>

**Why does it matter?**
- <bullet>

**Proposed fix**
<one or two sentences>

```
<code snippet>
```
```

## gh API command (inline comment)

```bash
gh api repos/<ORG>/<REPO>/pulls/<PR_NUMBER>/comments \
  --method POST \
  -f commit_id="<head-sha>" \
  -f path="<file>" \
  -F line=<N> \
  -f side="RIGHT" \
  -f body='<comment body>'
```

## Summary review

After all inline comments are posted, submit the review:

- Any blocking concern present → `gh pr review --request-changes`
- Non-blocking concerns only   → `gh pr review --comment`

Body: one or two plain sentences summarising the review. No severity labels.
