# Git authentication with multiple GitHub accounts on Linux and macOS

I use [GitHub CLI](https://cli.github.com/manual/) to setup authentication and
always clone with HTTPS.  I use this approach because:

1. GitHUB CLI (i.e., `gh`) is awesome
2. Setting up authentication with GitHUB CLI is easy: `gh auth login`
3. HTTPS is less likely to be blocked by firewalls than SSH

If you use SSH, or don't want to use GitHUB CLI, then this document is probably
not useful for you.

## What's The Problem?

On my macOS system, Git credentials (OAuth tokens, in this case) are stored in
Apple [Keychains](https://en.wikipedia.org/wiki/Keychain_(software)).  This
works great when you use a single GitHub account for all your work.  If you use
multiple accounts and use private repos, then it doesn't work well because some
software layer between you and the keychain fails to keep the tokens separate.

## My Approach on Linux

The solution on Linux wasn't obvious to me.  I found an approach that works for
me that I describe here.  My approach on macOS is the same, but requires extra
steps to keep Apple Keychain out of the picture.

My approach on Linux is to use different `gh` config dirs for different GitHub
accounts.  By default, `gh` config dir is `$HOME/.config/gh`.  I change that to
`$HOME/.config/gh-<GIT_ACCOUT_USERNAME>`, and then use env var `GH_CONFIG_DIR`
to distinguish between the accounts.

The following sections describe step-by-step how to set it up.

## Why do I need more than one GitHub account?

I have an account at work, provided by my employer.  I must use that account to
access work related repos.

I have my own account for personal use.  I use it at work because it has repos
with initialization files like like `.bashrc` and `.emacs.d/`, as well my
    personal treasure trove of shell scripts that I've come to rely on.

## Setup Details (Linux and macOS)

macOS version info:

| Software        | Version                | Command                        |
|:----------------|:-----------------------|:-------------------------------|
| macOS           | Sonoma 14.5            |                                |
| Xcode Git       | 2.39.3 (Apple Git-146) | /usr/bin/git --version         |
| Homebrew GH CLI | 2.52.0 (2024-06-24)    | /opt/homebrew/bin/gh --version |

Linux version info:

| Software | Version     | Command       |
|:---------|:------------|:--------------|
| OS       | 22.04       |               |
| Git      | 2.34.1      | git --version |
| GH CLI   | 2.4.0_dfsg1 | gh --version  |


### MacOS Only: Remove old keychain entries for `gh`


Remove old `gh` config dirs
```
rm -fr $HOME/.config/gh
rm -fr $HOME/.config/gh-*   # if you're repeating these instructions

```

### Cleanup up Environment Variables

```
env | grep GH
```

If you see `GH_CONFIG_DIR`, I recommend unsetting it and maybe removing it from your
environment setup scripts to avoid problems down the road.

```
unset GH_CONFIG_DIR
```

### Verify git config settings

Use this command to see all git config settings and where they came from:

```
git config --show-origin -l
```

We're most interested in settings related to credentials, so:

```
git config --show-origin -l | grep -i cred
```

On macOS, you might see this if you're using homebrew:

```
file:/opt/homebrew/etc/gitconfig        credential.helper=osxkeychain
```

Or this if using Xcode Git:

```
file:/Library/Developer/CommandLineTools/usr/share/git-core/gitconfig   credential.helper=osxkeychain
```

I edited those files and removed the setting.  It didn't stop git from using
Apple`s Keychain (I don't know why, is using Keychain a compiled-in behavior?).
So I don't think you can leave those settings be, and these instructions will
still work.

### For each GitHub account

Follow these steps for each GitHub account.  This example assumes the GiHub
account username is `foobar`.

#### Update git config

Add the following lines to `$HOME/.gitconfig`:

```
[credential]
  useHttpPath = true

[credential "https://github.com/foobar"]
  username = foobar
  helper = !GH_CONFIG_DIR=$HOME/.config/gh-foobar gh auth git-credential
```

If this is not the first GitHub username, you should omit `useHttpPath = true` (no need to repeat it).

#### Open browser and login to GitHub

Login to foobar's account: https://github.com/foobar

This makes the next step a bit easier

#### Use `git auth login` get credentials

On the command line:

```
GH_CONFIG_DIR=$HOME/.config/gh-foobar gh auth login
```

This sets the env var `GH_CONFIG_DIR` for this one command. But you knew that.

`gh auth login` will ask a bunch of questions.  Take the defaults.  Here's an example session:

```
What account do you want to log into?  **GitHub.com**
What is your preferred protocol for Git operations on this host? **HTTPS**
Authenticate Git with your GitHub credentials? **Yes**
How would you like to authenticate GitHub CLI? **Login with a web browser**
First copy your one-time code: 1AB0-869E
Press Enter to open github.com in your browser...
Authentication complete.
Configured git protocol
Logged in as foobar
```

On Linux, the security token is stored in `$GH_CONFIG_DIR/host.yml`.  For example:

```
github.com:
    git_protocol: https
    user: foobar
    oauth_token: gho_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

On macOS, it is stored in the keychain.  Storing multiple GitHub account tokens in the keychains
doesn't work for me.  So in the next step I move it from the keychain to the file.

### WARNING: Protect this token

If someone gets this token, they can login to GitHub and pretend to be you.  If
your home directory is on a shared file system, then think twice.  Don't keep
anything valuable or sensitve on that GitHub account.

### Linux Only: check to see if token is in the file

If it's not in the file, then... I don't know.  Cross your fingers and keep moving?

```
cat $HOME/.config/gh-foobar/hosts.yml
```

### macOS Only: move token from the keychain to GH_CONFIG_DIR

You can see the token with:

```
GH_CONFIG_DIR=$HOME/.config/gh-foobar gh auth token
```

You can append it to `$GH_CONFIG_DIR/hosts.yml`:

```
token=$(GH_CONFIG_DIR=$HOME/.config/gh-foobar gh auth token)
echo "    oauth_token: $token" >> $HOME/.config/gh-foobar/hosts.yml
```

Preview the file to ensure there are no extra blank lines and the indentation is correct:

Once the token is in the file, you can remove it from the keychain.

First, see if it's there:

```
security find-generic-password -s gh:github.com
```

Then delete:

```
security delete-generic-password -s gh:github.com
```

Repeat find/delete to make sure you get them all.  There are usually two: one
with the account name and one without.

You can also fire up the app to delete them:

```
Command-SPACE keychain open
Search "All Items" for "gh:"
Select each matching row and delete it
```

### Double check git config

On macOs, new credential entries kept showing up in `$HOME/.gitconfig`.

```
git config --show-origin -l | grep -i cred
```

I removed new entries.  Not sure who is adding them (probably `gh auth login`).

### Verify

Veify with `gh auth status`:

```
GH_CONFIG_DIR=$HOME/.config/gh-foobar gh auth status
```

Look for this output:
```
Logged in to github.com account foobar (/.../.config/gh-foobar/hosts.yml)
```

On macOS, if you see the following, then the token is not being retrieved from
the file.  The commands will work, but things will go awry if you add a second
GitHub user.

```
Logged in to github.com account foobar (keyring)
```

Use GH_CONFIG_DIR to set account for `gh` commands:
```
GH_CONFIG_DIR=$HOME/.config/gh-foobar gh repo list
GH_CONFIG_DIR=$HOME/.config/gh-foobar gh repo clone my-project
```

Once you clone it, you can use `git` and make use of the auth setup
without having to set `GH_CONFIG_DIR`.  For exmaple:

```
cd my-project
git pull
edit files
git add
git commit
git push
```


