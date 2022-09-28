+++
title = "Rename commits in Git"
date = 2022-09-27

[taxonomies]
tags = ["git", "rebase", "vscode"]
+++

Working on large pull requests can be hard. It often is even harder to review them.
A clean commit history can help reviewers out. Naming the commits consistently can help in that regard. Often it is hard to do this consistently beforehand, so sometimes it would be nice to rename commits after already committing them.

In this post I'll explain 3 methods of renaming commits:

- [Rename the last commit](#rename-the-last-commit) using `git commit --amend`
- [Rename commits one-by-one](#rename-commits-one-by-one) using `git rebase` and `reword`
- [Rename commits in batch](#rename-commits-in-batch) using `git rebase` and amend

<!-- more -->

## Rename the last commit

With Git it is possible to amend the latest commit. This can be done using:

```sh
git commit --amend --message "Your new commit message"
```

This can be useful, but only when doing so for a single (and latest) commit.

## Rename commits one-by-one

You can use `git-rebase` to mark which commits you'd like to rename and then rename them one-by-one.

For this we use the following command:

```sh
git rebase \
  --interactive \
  --fork-point origin/HEAD
```

Below I'll explain what each part of this command does.

### `git rebase` <!-- omit in toc -->

Rebase is a command in git to rewrite history. It can do this over a large amount of commits by replaying commits after each-other. Usually it 'picks' commits in the order of your history log, but it can do many other things.

See [git-rebase](https://git-scm.com/docs/git-rebase) for more information.

### `--interactive` <!-- omit in toc -->

Interactive rebase will open up an editor that allows you to change the commands that git will execute during its rebase. It preloads the editor with all commits that are going to be picked. When the editor opens it shows the commands git is going to execute. Often something like:

```txt
pick deadbee The oneline of this commit
pick fa1afe1 The oneline of the next commit
```

In the editor you can change these commands. Once you save and quit the editor, git will execute these commands in top-to-bottom order.

See [git-rebase --interactive](https://git-scm.com/docs/git-rebase#Documentation/git-rebase.txt---interactive) for more information.

By default, most distributions use `vi` or `nano` as their editor. You can configure this to your liking. I like to use vscode, which can be configured using:

```sh
git config --global sequence.editor "code --wait"
```

See [sequence.editor](https://git-scm.com/docs/git-rebase#Documentation/git-rebase.txt-sequenceeditor) for more information about configuring an interactive rebase editor.

### `--fork-point origin/HEAD` <!-- omit in toc -->

The commit reference `origin/HEAD` refers to the default branch of the repository. This depends on the age of the repository. Git used to default to `master` as its default branch, but nowadays `main` is the default. Because this changes from repository to repository, it helps to use `origin/HEAD` to refer to whichever is default in your repository.

The `--fork-point` takes an upstream branch as argument. So in this case `origin/HEAD`. It'll look up the point in history where your current branch (`HEAD`) forked off from the upstream branch (`origin/HEAD`). It'll pick all commits between that forked off commit until `HEAD`.

In terms of pull requests, the fork-point is the base of the pull request, thus rebase picks all commits that you have made in your pull request branch and none of the ones on the master or main branch.

To illustrate this a bit further consider the following:

```txt
a - b - c - d    <- origin/HEAD (= master branch)
     \
      e - f - g  <- HEAD (= your pull request branch)
```

`origin/HEAD` points to commit `d`. `HEAD` points to commit `g`. The `--fork-point origin/HEAD` will point to `b`.

`git rebase` will thus try to replay commits `e`, `f` and `g` on top of commit `b`.

### Tie it together <!-- omit in toc -->

When using the command:

```sh
git rebase \
  --interactive \
  --fork-point origin/HEAD
```

The editor will open up with the following contents:

```txt
pick deadbee The oneline of this commit
pick fa1afe1 The oneline of the next commit
```

Replace `pick` for each commit that you'd like to rename. Let's say we're going to rename all commits:

```txt
reword deadbee The oneline of this commit
reword fa1afe1 The oneline of the next commit
```

Save and exit the editor.

Rebase will execute these commands top-to-bottom. It'll open up the editor again for each commit and let you change the commit message of each commit.

This can be useful to change the message of a few commits in history, but once you want to change the commit message of a large number of them it helps to do this in one go. See below.

## Rename commits in batch

This method allows to change the commit message of a large number of commits in one go.

The command I use is very similar to the command of method B:

```sh
git rebase \
  --interactive \
  --fork-point origin/HEAD \
  --exec "git commit --amend --message ''"
```

The only addition here is the `--exec` option. See below.

### `--exec "git commit --amend --message ''"` <!-- omit in toc -->

In interactive rebase, when using the `--exec` option it _inserts_ the command after each `pick`:

```sh
git commit --amend --message ''
```

The resulting rebase commands shown in the editor will look like:

```txt
pick deadbee The oneline of this commit
exec git commit --amend --message ''
pick fa1afe1 The oneline of the next commit
exec git commit --amend --message ''
```

### Tie it all together <!-- omit in toc -->

When executing the above command, the editor will open showing something like:

```txt
pick deadbee The oneline of this commit
exec git commit --amend --message ''
pick fa1afe1 The oneline of the next commit
exec git commit --amend --message ''
```

Next, you can use your editor to change the messages of each commit. For instance, you can prefix the messages with `fix:` or `feat:`:

```txt
pick deadbee The oneline of this commit
exec git commit --amend --message 'fix: The oneline of this commit'
pick fa1afe1 The oneline of the next commit
exec git commit --amend --message 'feat: The oneline of the next commit'
```

I found multi-cursor features in vscode particularly useful for this. Using `Ctrl+D` it is possible to select all `pick` commands. With the multiple selections you can select the commit message and copy-paste it into the `--message` argument See [multiple selections](https://code.visualstudio.com/docs/editor/codebasics#_multiple-selections-multicursor) for more information.

Once finished, save and exit the editor and rebase will:

- Reset to the base of your pull request
- (Cherry-)pick the `deadbee` commit
- Amend the commit with the new message: "fix: The oneline of this commit"
- (Cherry-)pick the `fa1afe1` commit
- Amend the commit with the new message: "feat: The oneline of the next commit"

This leaves you with your pull request commits having been renamed in one go.
