# gitlab-push-mirror-probe

Nothing here is a real project. This repository exists to run one workflow
against itself.

`infra/gitlab/restore.sh` in the GitOps repo restores GitLab work back to GitHub
after an outage. It does not run `git` — it turns on GitLab CE's **push
mirroring** and lets GitLab do the pushing, on two claims read off GitLab's
source:

1. GitLab sends **LFS objects** before the push, so what arrives is not a tree
   of dangling pointers. `git push --all` by hand sends only the pointers.
2. `keep_divergent_refs=true` makes GitLab **leave a remote that has moved ahead
   alone and report it**, instead of quietly forcing over it.

Reading source is not the same as watching it happen. [`push-mirror.yaml`][wf]
watches it happen: it starts GitLab CE 19.3.1 inside the job, gives it branches,
a tag and an LFS object, arranges for one branch to have diverged on GitHub,
turns the mirror on, and then checks what actually arrived here.

[wf]: .github/workflows/push-mirror.yaml

## What it found

Both claims hold. The LFS object arrives whole, and a ref that had moved ahead
on GitHub was left exactly where it was.

One thing the source reading did not predict, and the runbook now has to say:

```json
{
  "update_status": "failed",
  "last_error": "Some refs have diverged and have not been updated on the remote:\n\nrefs/heads/probe/diverged",
  "last_successful_update_at": null
}
```

**`failed` does not mean nothing was delivered.** One held-back ref fails the
whole run, and `last_successful_update_at` stays `null`, while every other
branch and tag arrived. An operator who reads the status and stops there will
conclude the restore did not happen. The refs that were held back are named in
`last_error`, and that is the field to read.

## Why the workflow lives in the repository it pushes to

It needs **no personal access token**. The push comes from a GitLab that is
created and destroyed inside the job, and the credential it pushes with is the
job's own `GITHUB_TOKEN` — which is minted at the start of the job and revoked
at the end, and can write to **this repository only**. So the target has to be
this repository.

That is a better fit than a PAT, not a worse one. A push mirror stores its
credential in the URL, in GitLab's database, for as long as the mirror exists.
Here the database is thrown away with the container, minutes later.

## Warning

`refs/heads/probe/*` and `refs/tags/probe-*` are the job's scratch space. It
rewrites and deletes them.

Do not point this workflow at a repository that matters. With
`keep_divergent_refs=false` GitLab deletes remote branches the source does not
have — which here would include the branch holding the workflow.
