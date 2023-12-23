+++
title = 'Building a custom Emacs auth-source'
date = '2023-12-22'
tags = ['emacs', 'security', 'okta']
+++

My employer, [Okta](https://okta.com), has recently been making security
improvments to how we access all sorts of internal systems.  As part of that
hardending, we've been disallowed from using SSH keys and long-lived Github
tokens to access our code on GitHub.  In place of that, we've now got an
internal tool that grants us short-lived tokens on demand, after SSOing (through
Okta of course).

This is, in general, a good idea, and means that even if you gain access to my
machine somehow you won't automatically have my privileges to access our
codebase. The way it integrates with standard `git` tooling is interesting and
uses a git subsystem I wasn't previously aware of: the [git credential
helper](https://git-scm.com/docs/gitcredentials).

This mostly Just Works™, which is pretty great.  My main interaction with our
repos is via the wonderful `magit` package in Emacs, and although I'm one of the
very few Emacs users in our company I didn't have to do anything special to keep
working away.

However, what didn't work right away was [forge](https://magit.vc/manual/forge),
another package by the author of `magit`, which lets you create and edit PRs and
otherwise deal with the parts of GitHub (or other forges like Gitlab etc.) that
aren't part of git proper.

So, this being Emacs, I set out to fix it.

## How does this tool work anyway?

I've had to fix a few other of our internal tools that interact with git or
GitHub, so I had already figured out how our new token-granting tool works.  It
turns out this is [documented quite thoroughly](https://git-scm.com/docs/git-credential "git-credential").

When git tooling needs credentials to communicate with a remote (for example to
`push`, `pull`, etc.), it checks to see if there's a `credential.helper` in
the config and executes it to retrieve credentials.  When you set up Okta's new
internal tool, it adds a line in your `.gitconfig` like this:

```
[credential]
   helper = "/home/emoses/awesome-okta-tool --some options"
```

When `git` or any other system needs a token, it executes
`/home/me/awesome-okta-tool --some options get`, and then the helper reads
off of stdin, looking for input like this:

```
protocol=https
host=github.com
path=okta/imporantrepo
username=emoses

```

And then goes off and does it's thing (including getting 2FA via Okta), and
responds with

```
protocol=https
host=github.com
path=okta/coolrepo
username=emoses
password=gh_shorttermtoken1
```

## So why doesn't forge work?

The `forge` manual has a [whole
section](https://magit.vc/manual/ghub/Getting-Started.html#Getting-Started)[^1]
about setting up tokens and storing them so `forge` can get it back out.  It
uses `auth-source`, an Emacs built-in package that's provides an API that wraps
a bunch of different way to store login credentials, including old-school plaintext
`.netrc` files, GPG-encrypted `authinfo.gpg` files, and APIs like the unix
`secrets` API or the MacOS Keychain.  Notably it does *not* use the git
credential helper, presumably because your git credentials (e.g. an SSH key used
to push to git remotes) aren't usually the same type of thing as the API token
used by a forge.

I'd followed the instructions to put a long-lived GitHub token with the right
username and host in the MacOS keychain, and `forge` was happily retrieving that
token using `auth-source`.  But now, of course, those long-lived tokens are no
longer allowed.

## Let's extend auth-source

`auth-source` is a generic façade, meant to retrieve tokens from all sorts of
different auth backends.  So how about we just write a new backend?
Since communicating with the git credential helper is pretty straightfoward, how hard
could that be?

### Kinda hard, actually

Turns out auth-source has basically no developer documentation for writing a new
backend.  But this is Emacs!  Let's dig in!

### How does it work with the keychain?

I'd messed with `auth-source` a bit before[^2] and knew that you could retreive a secret by calling `(auth-source-search
:user USERNAME :host HOST)`. The sources that it uses are defined by a list called `auth-sources`, which looks like
`(macos-keychain-generic macos-keychain-internet "~/.authinfo.gpg" "~/.authinfo" "~/.netrc")`.  So I took a look at {{< keyseq >}}C-h v auth-source{{< /keyseq >}} and followed it to the [source](https://github.com/emacs-mirror/emacs/blob/emacs-29.1/lisp/auth-source.el#L225).  This lays out all the options for auth sources, but didn't tell me what I wanted to know, so I browsed through the file some more looking for `macos-keychain`, and landed on `auth-source-macos-keychain-search`.  This is the actual function that does the searching, but where's it refereneced?  Searching for it in the file we find [this snippet](https://github.com/emacs-mirror/emacs/blob/emacs-29.1/lisp/auth-source.el#L413):

```elisp
(defun auth-source-backends-parser-macos-keychain (entry)
  ;; take macos-keychain-{internet,generic}:XYZ and use it as macOS
  ;; Keychain "XYZ" matching any user, host, and protocol
  ;; yadda yadda yadda, check the name of the entry [yaddas mine]
      (auth-source-backend
       (format "Mac OS Keychain (%s)" source)
       :source source
       :type keychain-type
       :search-function #'auth-source-macos-keychain-search
       :create-function #'auth-source-macos-keychain-create)))))

(add-hook 'auth-source-backend-parser-functions #'auth-source-backends-parser-macos-keychain)
```

Now we're getting somewhere!  This function looks like it takes the entry in `auth-sources` and returns an `auth-source-backend` if the entry is a string or symbol that looks like `'macos-keychain-generic`, and it has a `:search-function` that searches the keychain.  Let's copy-paste some code and see where it gets us.



[^1]: This is actually a link to the `ghub` documentation, a related package, which is referenced
    by the `forge` documentation.

[^2]: Getting `forge` to work the first time, actually.
