+++
title = "Autosquash in Git"
date = 2022-09-28

[taxonomies]
tags = ["git"]
+++

Keeping commit history in pull requests clean can be hard. `git-rebase` offers some methodologies to alter history, but often this results in a clunky workflow.

Autosquash is a feature of `git-rebase` that I have recently grown fond of. It can be used to more easily alter commits in history. In this post I'll give some insight how autosquash can be used in practice.

<!-- more -->

## What is autosquash?

Interactive rebase is a feature in Git that allows you to interactively change the rebase instructions that Git would normally execute automatically.
When using this feature, you can change or reorder instructions so that you can have a clean history for your branch.
With autosquash, commit messages act like instructions for interactive rebase that are automatically inserted in the right places.

To give an example, say we have the following history:

```console
$ git log --oneline
dbd9e8a Introduced a new feature
e9ca88e Refactored some code
```

We have some changes staged that were meant to be part of the refactoring (`e9ca88e`). We can do the following:

```sh
git commit --message "fixup! Refactored some code"
```

The history will look like:

```console
$ git log --oneline
ad95d7d fixup! Refactored some code
dbd9e8a Introduced a new feature
e9ca88e Refactored some code
```

We can use interactive rebase with autosquash to automatically mark `ad95d7d` as a fixup for `e9ca88e`:

```sh
git rebase \
  --interactive \
  --fork-point origin/HEAD \
  --autosquash
```

This opens up the editor to allow you to change the sequence. You'll notice `ad95d7d` is placed right below `e9ca88e` with a fixup command:

```txt
pick e9ca88e Refactored some code
fixup ad95d7d fixup! Refactored some code
pick dbd9e8a Introduced a new feature
```

Upon saving and exiting the editor rebase will combine the two commits `ad95d7d` and `e9ca88e`:

```console
$ git log --oneline
e10c4dd Introduced a new feature
9ac5ef3 Refactored some code
```

Note that both commits have changed. This is what autosquash does: for interactive rebases it automatically places prefixed (`fixup!`) commit messages below its matching commit.

## Fixup commits

In practice it can be more practical to refer to a specific commit instead of copy-pasting a title from your log. This is done with the `--fixup` option for `git-commit`:

```sh
git commit --fixup=<revision>
```

In the above example, instead of commiting using the `fixup!` message, it could've been handled using:

```sh
git commit --fixup=e9ca88e
```

This automatically picks the title of `e9ca88e` and prefixes it with `fixup!`.

## Amend commits

Next to fixup commits, it is also possible to amend previous commits. Say in our previous example we not just wanted to fix the contents of the previous commit, but also change the commit message. We could've done so using:

```sh
git commit --message 'amend! Refactored some code

Refactored some code and more'
```

Here, the first line of the message indicates which commit to amend, followed by a blank line, followed by the new message.

When executing interactive rebase again, it'll show:

```txt
pick ad95d7d Refactored some code
fixup -C 9f6fb29 amend! Refactored some code
pick e9ca88e Introduced some feature
```

After closing the editor, the history will look like:

```console
$ git log --oneline
f3b938f Introduced some feature
aff88ee Refactored some code and more
```

Notice that the amend commit has been combined with `ad95d7d` and the new commit message was used.

A shorthand to create such commits can also be found in `git-commit`:

```sh
git commit --fixup=amend:<revision>
```

When executing this, the editor will open with a preloaded commit message:

```txt
amend! Refactored some code

Refactored some code
```

Here you can edit the third line to use the new message upon the next autosquash.

## Reword commits

Whenever you'd want to change a commit in history you could use the above `--fixup=amend:` option to change the title. However, in this form `git-commit` doesn't allow you to do this without staging any changes. This is where `--fixup=reword:` can be used.

```sh
git commit --fixup=reword:<revision>
```

It'll work without staging anything, but it will also open up the editor with the following preloaded commit message:

```txt
amend! Refactored some code

Refactored some code
```

Note that this _also_ creates an `amend!`-prefixed commit message. It will do exactly the same as an amend commit, but because this commit will have an empty diff, it'll only alter the message upon the next autosquash.

## Stacking commits

Interactive rebase using autosquash will look at _all_ prefixed commits in the range of rebase. It'll place the right commands in the order of your history. Multiple fixup commits can be applied on the same 'picked' commit.

It allows you to keep focused while working on a branch. When you're done with your branch and want to publish it for review, you can apply autosquash.

Often when I worked on a branch this way, I'm not that interested in viewing or editing the interactive rebase sequence of commands. To avoid opening the editor I use:

```sh
git rebase \
  -c sequence.editor=true \
  --interactive \
  --fork-point origin/HEAD \
  --autosquash
```

Here the sequence editor is set to `true` during the execution of the command. `true` in this case refers to the command `true`, thus interactive rebase will call the `true` command instead of your editor. It will immediately start processing the rebase commands.

I use this method to clean up my pull requests. Therefore I have aliased this `pr-clean`, similar to [other `pr-*` related git aliases](https://github.com/bobvanderlinden/nixos-config/blob/2e906850209e10ace96695967247060e8c76c24b/home/default.nix#L308-L313).
