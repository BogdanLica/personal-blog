---
title: Notes on Git
date: 2019-06-02 16:05:23
tags:
---
##  High Level Overview
### Usage: track changes of files over time

* Git tracks different snapshots of a project by using **commits**
* a **branch** is a pointer to a commit in the repo
* **HEAD** is the symbolic name for the currently checked out commit
* **detached HEAD** means that the HEAD is poiting to a commit instead of a branch

#### :nerd_face: reference  : git can resemble a big directed acyclic graph


## Basic commands

* ### Local

| Command | Description |
----------|--------------
| **git init example** | Initialise a repository with the name "example"|
| **git status**| See the files in the staging area compared to the current branch |
| **git add myfile** | Add a file to the staging area|
| **git rm myfile** | Remove a file from the working index and staging area|
| **git commit -m "my message"** | Save all the stages file as a new commit in the current branch with the message "my message"|
| **git diff** | Shows what changes have been made that are not yet committed(compares the working area, **not the staging area** with the branch) |
| **git diff --staged** | Shows the changes that have been staged, but not yet committed|


* ### Remote

| Command | Description |
|---------|-------------|
| **git pull remote** | **Fetch** the new changes from the remote and **merge** them to the current branch|
| **git clone url_to_repo** | Copy a repo to the current directory and save it with the same name as the repo|


##### Visual Representation
![](/../posts_resources/Notes-on-Git/git_workflow.png)



## Intermediate commands

* ### Local

| Command | Description |
|---------|-------------|
| **git mv** | Rename a file and detect it automatically in the staging area |
| **git diff --staged**| Show any changes that have been made in the staging area compared with the branch|
| **git reset HEAD~1** | Move the current branch 1 commit back |
| **git branch -f master HEAD~2**| Move the "master" branch to point 2 commits back from where HEAD points to |
| **git bisect start** | Find the first bad commit using binary search |
| **git cherry-pick <Commit_1> <Commit_2>** | Copy the commits(as hashes) on top of HEAD |
| **git checkout sha256_hash**| Move the HEAD to the commit specified (**WARNING**: This results in a detached HEAD if a branch does not point to that commit) |
| **git rebase -i HEAD~4** | From where currently are, go back 4 commits and do rebasing in an interactive way |


* ### Remote

| Command | Description |
|---------|-------------|
| **git pull -r** | Same as **git pull**, but do rebasing|
|**git revert HEAD~1**| Move the remote branch 1 commit back |




## When production is on :fire:

| Command | Description |
| --------|------------ |
| **git diff HEAD~2 HEAD**| See the changes between the commit at the current pointer and 2 commits behind the pointer |
| **git log --oneline** | See the message from each commit |
| **git log -p -n 2**| See the message from the last 2 commits(HEAD and HEAD~1) commit and the differences|
| **git bisect start**| Find the first bad commit|
| **git blame <file>**| Show the last author who made changes on each line of a file|


##### Note:

* Caret
  * **master^**
    * the first parent of master(moves up by one)
* '~' operator
  * **HEAD~2**
    * 2 commits up from the HEAD


## Advance knowledge

### Internals

#### Filesystem
* a git repository is just the folder ".git"
* ignored files can be specified in the ".gitignore" file
* git does not store the diff, but if make another copy of each object
  * storing the diff would mean that when you try to get the up to date version of a repo, all "n" commits would have to be computed while git only does O(1)
* objects stored by git:
  * Blob(binary large objects) -> contains the data itself(source code,image,video etc)
  * Tree -> pointer to file names, other trees
  * Commit -> author, message, pointer to a tree
  * Tag


#### Commands

* **git add** creates Blob objects
* **git commit** creates Tree and Commit objects
* see the content of an object :


```shell
> git show e7d8e73a
```

```shell
> git show --pretty-print=raw e7d8e73
```
* see the content of a tree object:


```shell
> git ls-tree f9e2e3ed
```
Note: the whole hash is not needed, only the shortest unique identifier (usually 8 characters)



## Where to practice?
* [Learn Git branching](https://learngitbranching.js.org/)
* [Git exercises](https://gitexercises.fracz.com/)
* [Katakoda](https://www.katacoda.com/courses/git)
* [More exercises](https://training-course-material.com/training/Git_exercises)

## Further reading
* [Using Git](https://code-maven.com/slides/git/)