---
layout: post
title:  When your GitHub contribution not showing
categories: [GitHub]
tags: [GitHub, CI/CD]
description: Ensure your GitHub email is correct for contributions, and use a script to fix past commits.
---

I've been working hard to push contributions to my GitHub, but they aren't being recorded at all. 

So I looked for the reason.

The reason is that I had been pushing the code from Sourcetree to Github, but the email registered as the author in SourceTree was different from my GitHub email, so it was not recognized.

So I will try to explain two things in this article through my experience.

1. How to change the repository author account to GitHub email
2. How to change the author of commits that have already been pushed incorrectly

---

## How to change the repository author account to GitHub email

1. Check the registered email in GitHub settings > Emails tab.

2. Open the command window in the folder you are working in and type the command below.
```bash
git config --list
```
Then, the setting values will appear as shown in the picture below, and there will be email information. (It may be different from your GitHub account email)
![20240723_gitConfig_list.png](assets/img/posts/240916/20240723_gitConfig_list.PNG)

3. Change email value settings using the commands below. 
```bash
git config user.email "Your Github email address"
```
However, this only changes the folder in the current directory.

If you want to change the email address for all local repositories, you can use the command below.
```bash
git config --global user.email "Your email address"
```

If the contributions are not pushed properly even after changing the email, you need to check the name and change it to be the same as Github.
```bash
git config user.name "Your Github name"
```

Then future commits will be properly registered as contributions.

So how should we change the commits we push previoulsy?

---

## How to change the author of commits that have already been pushed incorrectly

There is a way to remember the hash codes of previous commits and change them through rebase, but in this case, you have to do each commit yourself.

However, when there were dozens of things that needed to be modified, I was worried about whether I should do them one by one, and like a programmer, I created a script.
(Please note that you must run it from each repository command window.)

``` bash
# git filter-branch: Use Git's filter-branch command to rewrite all commits on the branch.
git filter-branch --env-filter '

# OLD_EMAIL: Specify the incorrect email address.
OLD_EMAIL="old@example.com"

# CORRECT_NAME: Specify the correct name for the commits.
CORRECT_NAME="Your Correct Name"

# CORRECT_EMAIL: Specify the correct email address for the commits.
CORRECT_EMAIL="correct@example.com"

# Check if GIT_COMMITTER_EMAIL matches OLD_EMAIL.
if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    # If it matches, set the committer name and email to the correct values.
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi

# Check if GIT_AUTHOR_EMAIL matches OLD_EMAIL.
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    # If it matches, set the author name and email to the correct values.
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi

' --tag-name-filter cat -- --branches --tags

# --tag-name-filter cat: Do not perform tag name filtering. Keep tag names as they are.
# -- --branches --tags: Apply the filter to all branches and tags.

```
After running the above script, force push the changed commit history.

If you check now, previous commits will be properly registered in contributions.
