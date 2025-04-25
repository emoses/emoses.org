+++
title = 'Git One Liners: Bulk-Amending a Branch'
date = 2025-04-25
categories = ['blog']
tags = ['git', 'tips']
summary = 'I made a mistake with my git config recently, and ended up creating all the commits in a branch with the wrong email address.  I can fix it up with a neat one-liner.'
+++

I made a mistake with my git config recently (I've updated {{< reftitle "keeping-work-separate" >}} to avoid that
mistake 😬), and ended up creating all the commits in a branch with the wrong email address.  The also should have been
signed and they weren't.

Here's how to fix that in one go:

```bash
git rebase \
--exec 'git commit --amend --no-edit -n --gpg-sign --author="Evan Moses <the.correct@email.address>"' \
$(git merge-base origin/main HEAD)
```

## Breaking it down

```bash
git rebase # Rebase some commits
```
```bash
--exec # And execute the following command for each commit
```
```bash
git commit --amend --no-edit -n # Amend the commit without changing the commit message,
                                # contents, or running any git hooks
```
```bash
--gpg-sign --author="..."` # This is the bit I actually wanted to do differently
```
```bash
$(git merge-base origin/main HEAD) # This is a useful command to find the "branch point"
                                   # where your current branch diverges from main.
```

So: this will iterate over all the commits on my current branch and amend them.  Done.
