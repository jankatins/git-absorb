# git absorb

This is a port of Facebook's [`hg absorb`](https://www.mercurial-scm.org/repo/hg/rev/5111d11b8719), which I first read about on [mozilla.dev.version-control](https://groups.google.com/forum/#!msg/mozilla.dev.version-control/nh4fITFlEMk/ZNXgnAzxAQAJ).

## Elevator Pitch

You have a feature branch with a few commits. Your teammate reviewed the branch and pointed out a few bugs. You have fixes for the bugs, but you don't want to shove them all into an opaque commit that says `fixes`, because you believe in atomic commits. Instead of manually finding commit SHAs for `git commit --fixup`, or running a manual interactive rebase, do this:

```
git add $FILES_YOU_FIXED
git absorb --and-rebase
```

`git absorb` will automatically identify which commits are safe to modify, and which staged changes belong to each of those commits. It will then write `fixup!` commits for each of those changes.

With the `--and-rebase` flag, these fixup commits will be automatically integrated into the corresponding ones. Alternatively, you can check its output manually if you don't trust it, and then fold the fixups into your feature branch with git's built-in [autosquash](https://git-scm.com/docs/git-rebase#Documentation/git-rebase.txt---autosquash) functionality:

```
git add $FILES_YOU_FIXED
git absorb
git log # check the auto-generated fixup commits
git rebase -i --autosquash master
```

## Installing

The easiest way to install `git absorb` is to download the latest CI artifact from the actions tab. Artifacts are available for Windows, MacOS and Ubuntu.

Alternatively, `git absorb` is available in the following system package managers:

| Repository                  | Command                                      |
| --------------------------- | -------------------------------------------- |
| Arch Linux                  | `pacman -S git-absorb`                       |
| Debian                      | `apt install git-absorb`                     |
| DPorts                      | `pkg install git-absorb`                     |
| FreeBSD Ports               | `pkg install git-absorb`                     |
| Homebrew and Linuxbrew      | `brew install git-absorb`                    |
| nixpkgs stable and unstable | `nix-env -iA nixpkgs.gitAndTools.git-absorb` |
| Ubuntu                      | `apt install git-absorb`                     |
| Void Linux                  | `xbps-install -S git-absorb`                 |
| GNU Guix                    | `guix install git-absorb`                    |

## Compiling from Source

[![crates.io badge](https://img.shields.io/crates/v/git-absorb.svg)](https://crates.io/crates/git-absorb)

You will need the following:

- [cargo](https://github.com/rust-lang/cargo)

Then `cargo install git-absorb`. Make sure that `$CARGO_HOME/bin` is on your `$PATH` so that git can find the command. (`$CARGO_HOME` defaults to `~/.cargo`.)

Note that `git absorb` does _not_ use the system libgit2. This means you do not need to have libgit2 installed to build or run it. However, this does mean you have to be able to build libgit2. (Due to [recent changes](https://github.com/alexcrichton/git2-rs/commit/76f4b74aef2bc2a54906ddcbf7fbe0018936a69d) in the git2 crate, CMake is no longer needed to build it.)

Note: `cargo install` does not currently know how to install manpages ([cargo#2729](https://github.com/rust-lang/cargo/issues/2729)), so if you use `cargo` for installation then `git absorb --help` will not work. Here is a manual workaround, assuming your system has a `~/.local/share/man/man1` directory that `man --path` knows about:

```
wget https://raw.githubusercontent.com/tummychow/git-absorb/master/Documentation/git-absorb.1
mv git-absorb.1 ~/.local/share/man/man1
```

## Usage

1. `git add` any changes that you want to absorb. By design, `git absorb` will only consider content in the git index (staging area).
2. `git absorb`. This will create a sequence of commits on `HEAD`. Each commit will have a `fixup!` message indicating the message (if unique) or SHA of the commit it should be squashed into.
3. If you are satisfied with the output, `git rebase -i --autosquash` to squash the `fixup!` commits into their predecessors. You can set the [`GIT_SEQUENCE_EDITOR`](https://stackoverflow.com/a/29094904) environment variable if you don't need to edit the rebase TODO file.
4. If you are not satisfied (or if something bad happened), `git reset --soft` to the pre-absorption commit to recover your old state. (You can find the commit in question with `git reflog`.) And if you think `git absorb` is at fault, please [file an issue](https://github.com/tummychow/git-absorb/issues/new).

## How it works (roughly)

`git absorb` works by checking if two patches P1 and P2 *commute*, that is, if applying P1 before P2 gives the same result as applying P2 before P1.

`git absorb` considers a range of commits ending at HEAD. The first commit can be specified explicitly with `--base <ref>`. By default the last 10 commits will be considered (see [Configuration](#configuration) below for how to change this).

For each hunk in the index, `git absorb` will check if that hunk commutes with the last commit, then the one before that, etc. When it finds a commit that does not commute with the hunk, it infers that this is the right parent commit for this change, and the hunk is turned into a fixup commit. If the hunk commutes with all commits in the range, it means we have not found a suitable parent commit for this change; a warning is displayed, and this hunk remains uncommited in the index. 

## Configuration

### Stack size

When run without `--base`, git-absorb will only search for candidate commits to fixup within a certain range (by default 10). If you get an error like this:

```
WARN stack limit reached, limit: 10
```

edit your local or global `.gitconfig` and add the following section

```ini
[absorb]
    maxStack=50 # Or any other reasonable value for your project
```

## TODO

- implement force flag
- implement remote default branch check
- add smaller force flags to disable individual safety checks
- stop using `failure::err_msg` and ensure all error output is actionable by the user
- slightly more log output in the success case
- more tests (esp main module and integration tests)
- document stack and commute details
- more commutation cases (esp copy/rename detection)
- don't load all hunks in memory simultaneously because they could be huge
- implement some kind of index locking to protect against concurrent modifications
