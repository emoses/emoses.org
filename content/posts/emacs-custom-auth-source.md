+++
title = 'Building a custom Emacs auth-source'
date = '2023-12-22'
tags = ['emacs', 'security', 'okta']
draft = true
+++

My employer, [Okta](https://okta.com), has recently been making security
improvements to how we access all sorts of internal systems.  As part of that
hardening, we've been disallowed from using SSH keys and long-lived GitHub
tokens to access our code on GitHub.  In place of that, we've now got an
internal tool that grants us short-lived tokens on demand, after SSOing (through
Okta of course).

This is a good idea even if it adds a little friction, and means that even if
you gain access to my machine somehow you won't automatically have my privileges
to access our code base. The way it integrates with standard `git` tooling is
interesting and uses a git subsystem I wasn't previously aware of: the [git
credential helper](https://git-scm.com/docs/gitcredentials).

This mostly Just Works™, which is pretty great.  My main interaction with our
repos is via the wonderful `magit` package in Emacs, and although I'm one of the
very few Emacs users in our company I didn't have to do anything special to keep
working away.

However, another tool, [forge](https://magit.vc/manual/forge), needed more work.
It's another package by the author of `magit`, which lets you create and edit
PRs and otherwise deal with the parts of GitHub (or other forges like Gitlab
etc.) that aren't part of git proper.

So, this being Emacs, I got out my clippers and went yak shaving.

## How does this tool work anyway?

I've had to fix a few other of our internal tools that interact with git or
GitHub, so I had already figured out how our new token-granting tool works.  It
turns out this is [documented quite thoroughly](https://git-scm.com/docs/git-credential "git-credential").

When git tooling needs credentials to communicate with a remote (for example to
`push`, `pull`, etc.), it checks to see if there's a `credential.helper` in
the config and executes it to retrieve credentials.  When you set up Okta's new
internal tool, it adds a line in your `.gitconfig`:

```toml
[credential]
   helper = "/home/emoses/awesome-okta-tool --some options"
```

When `git` or any other system needs a token, it executes
`/home/emoses/awesome-okta-tool --some options get`, and then the helper reads
off of stdin, expecting input like this:

```
protocol=https
host=github.com
path=okta/imporantrepo
username=emoses

```

And then goes off and does its thing (including MFA via Okta), and responds with

```
protocol=https
host=github.com
path=okta/importantrepo
username=emoses
password=gh_shorttermtoken1
```

## So why doesn't 'forge' work?

The `forge` manual has a [whole
section](https://magit.vc/manual/ghub/Getting-Started.html#Getting-Started)[^1]
about setting up tokens and storing them so `forge` can get it back out.  It
uses `auth-source`, an Emacs built-in package that's provides an API that wraps
a bunch of different ways to store login credentials, including old-school plaintext
`.netrc` files, GPG-encrypted `authinfo.gpg` files, and APIs like the Unix
`secrets` API or the MacOS keychain.  Notably it does *not* use the git
credential helper, presumably because your git credentials (e.g. an SSH key used
to push to git remotes) aren't usually the same type of thing as the API token
used by a forge.

[^1]: This is a link to the `ghub` documentation, a related package, which is referenced
    by the `forge` documentation.

I'd followed the instructions to put a long-lived GitHub token with the right
username and host in the MacOS keychain, and `forge` was happily retrieving that
token using `auth-source`.  But now, of course, those long-lived tokens are no
longer valid.

## Let's extend auth-source

`auth-source` is a generic façade, meant to retrieve tokens from all sorts of
different auth backends.  So how about we just write a new backend?
Since communicating with the git credential helper is pretty straightforward, how hard
could that be?

### Kinda hard, actually

Turns out auth-source has basically no developer documentation for writing a new
backend.  But this is Emacs!  Let's dig in!

### How does it work with the keychain?

I'd messed with `auth-source` before[^2] and knew that you could retrieve a
secret by calling `(auth-source-search :user USERNAME :host HOST)`. The sources
that it uses are defined by a list called `auth-sources`, which looks like
`(macos-keychain-generic macos-keychain-internet "~/.authinfo.gpg" "~/.authinfo"
"~/.netrc")`.  So I took a look at {{< keyseq >}}C-h v auth-source{{< /keyseq >}}
and followed it to the
[source](https://github.com/emacs-mirror/emacs/blob/emacs-29.1/lisp/auth-source.el#L225).
This lays out all the options for auth sources, but didn't tell me what I wanted
to know, so I browsed through the file some more looking for `macos-keychain`,
and landed on `auth-source-macos-keychain-search`.  This is the actual function
that does the searching, but where's it referenced?  Searching for it in the
file we find [this function](https://github.com/emacs-mirror/emacs/blob/emacs-29.1/lisp/auth-source.el#L413):

[^2]: Getting `forge` to work the first time, actually.

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

This is promising! The function looks like it takes the entry in `auth-sources` and returns an `auth-source-backend` if the entry is a string or symbol that looks like `macos-keychain-generic`, and it has a `:search-function` that searches the keychain.  Let's copy-paste some code and see where it gets us.

```elisp
   (defun auth-source-git-credential-helper-search (&rest TODO)
      "TODO"
      )

  (defun auth-source-backend-git-credential-helper (entry)
    (when (and (stringp entry) (string-match "^git-credential-helper" entry))
      (auth-source-backend
       :source  "git-credential-helper"
       :type 'git-credential
       :search-function #'auth-source-git-credential-helper-search)))

  (add-hook 'auth-source-backend-parser-functions #'auth-source-backend-git-credential-helper)
```

So what does `auth-source-git-credentials-search` need to do?  It takes in a `spec`, which is the plist that `auth-source-search` takes, and returns a plist that looks like `(:user USERNAME :host HOSTNAME :type 'git-credentials :secret SECRET)`, where `SECRET` is either the secret itself or a function that evaluates to the secret.

With a bit more copy-paste we have something like this:

```elisp
(cl-defun auth-source-git-credential-helper-search (&rest spec
                                                            &key create delete type
                                                            &allow-other-keys)
    (cl-assert (not create) nil "git-credential-helper doesn't support create")
    (cl-assert (not delete) nil "git-credential-helper doesn't support delete")

    (when (string-equal (plist-get spec :host) "api.github.com")
      ;; forge appends "^forge" to the username, so get just the part before the ^
      (let ((user (car (string-split (plist-get spec :user) "\\^")))
            ;; Get the git credential helper from the config
            (helper (ignore-errors (car (process-lines "git" "config" "credential.helper")))))
        (when (and user helper)
          ;; call the helper and parse its output
          ))))
```

{{< aside >}}
#### Pro tip: don't use cl-defun when developing

Something I learned while debugging is that `cl-defun` seems to mess with
`edebug`.  Often when I'm doing exploratory coding I'll evaluate a function with
{{< keyseq >}}C-u C-M-x {{< /keyseq >}}, which enters the `edebug` debugger when
the function is called.  But with `cl-defun` this doesn't work right, and I
ended up just changing it to a normal `(defun auth--search (&rest spec) ...)`.
I didn't dig in to why this is the case, if I'm doing something wrong please let
me know.  {{< /aside >}}

### Let's get parsing

We have to actually run the credential helper and get the secret back from it.
Dealing with processes and pipes in Emacs was new to me.  I had used
`process-lines` to run a process and get its output back, but `process-lines`
can't pipe data to the process' stdin.  The basic idea is that everything in Emacs is a
buffer, and you use normal text-editing and buffer-navigation functions to deal
with it: you put the data you want to send in a string or a buffer, send it to
the process, and then the output of the process is written back to the buffer.

So, I'm gonna need a new buffer to collect the output, I'll need to call the
helper function with the right args, and then process the output.  We'll use
[`call-prcoess-region`](https://www.gnu.org/software/emacs/manual/html_node/elisp/Synchronous-Processes.html#index-call_002dprocess_002dregion),
which can take a string as input:

```elisp
(cl-defun auth-source-git-credential-helper-search (&rest spec
                                                            &key create delete type
                                                            &allow-other-keys)
    (cl-assert (not create) nil "git-credential-helper doesn't support create")
    (cl-assert (not delete) nil "git-credential-helper doesn't support delete")

    (when (string-equal (plist-get spec :host) "api.github.com")
      (let ((user (car (string-split (plist-get spec :user) "\\^")))
            (helper (ignore-errors (car (process-lines "git" "config" "credential.helper")))))
        (when (and user helper)
          (let (;; The command and args need to be in a list, and we need to add
                ;; the argument "get" to the end of the list
                (helper-and-args (append (string-split helper " ") (list "get")))
                ;; Create a temp buffer to process the output
                (out-buf (generate-new-buffer "output")))
            ;; unwind-protect works like a try/finally, allowing us to clean up
            ;; the temp buffer if there's an error.
            (unwind-protect
                (progn
                  ;; the first two args are the beginning and end point of a
                  ;; region to send, or a string and nil to send a string, which
                  ;; is what we'll do.
                  (apply #'call-process-region
                    (format "username=%s\nprotocol=https\nhost=github.com\n\n" user)
                    nil
                    (car helper-and-args)  ;; program name
                    nil                    ;; DELETE, we'll ignore
                    out-buf                ;; Destination buffer
                    nil                    ;; DISPLAY, we'll ignore
                    (cdr helper-and-args)) ;; The rest of the args
                  (with-current-buffer out-buf
                    ;; process the output
                    (let ((processed (auth-source-git-credential-helper--process-output)))
                      ;; If we've got a secret from the output, return it along
                      ;; with a :type property we got from the input spec
                      (and (plist-get processed :secret) (list (plist-put processed :type type))))))
              (kill-buffer out-buf)))))))
```

And then there's the function to process the output and return it as the plist
that `auth-source` expects.  A couple interesting things here: we'll take a page
from the keychain code and return a function that returns the secret, rather
than the secret as a string. This defends against an attacker that might be able
to access memory from another program or a core dump, although I'm not convinced
this is actually a useful layer of hardening, since the "decryption key" is also
in memory.  Also: we have to concatenate the host and protocol from the output
into the `:host` value in the result, so there's a little extra work there.  For
the actual parsing, we use `looking-at`, which tests the text at the point
against a regex (setting the `match` as other Emacs regexp functions do, see
[The Match
Data](https://www.gnu.org/software/emacs/manual/html_node/elisp/Match-Data.html)
for more information), and a small helper function to build up the result plist.

```elisp
(defun auth-source-git-credential-helper--append (result key &optional filter)
    "Append the value between match-end and the end of the line to
plist RESULT with KEY.  If FILTER is present, call it with the
match data and any existing result at that key, and put its value
at KEY."
    (let* ((data (buffer-substring-no-properties (match-end 0) (line-end-position)))
           (data (if (functionp filter) (funcall filter data (plist-get result key)) data)))
      (plist-put result key data)))

(defun auth-source-git-credential-helper--process-output ()
    (let ((ret '()))
      ;; Start at the beginning
      (goto-char (point-min))
      ;;Loop until we're at the end
      (while (not (eobp))
        (cond
         ;; We found password
         ((looking-at "^password=")
          (setq ret (auth-source-git-credential-helper--append
                     ret :secret
                     ;; Note: for this to work lexical-binding must be t
                     (lambda (data &rest _)
                       (let ((v (auth-source--obfuscate data)))
                         (lambda () (auth-source--deobfuscate v)))))))
         ((looking-at "^username=")
          (setq ret (auth-source-git-credential-helper--append
                     ret :user)))
         ((looking-at "^host=")
          (setq ret (auth-source-git-credential-helper--append
                     ret :host
                     ;; If we've already got protocol, append host
                     (lambda (data &optional existing) (concat existing data)))))
         ((looking-at "^protocol=")
          (setq ret (auth-source-git-credential-helper--append
                     ret :host
                     ;; If we've already got host, prepend protocol with ://
                     (lambda (data &optional existing) (concat data "://" existing))))))
        (forward-line))
      ret))
```

Let's test it.  We can make a buffer with the expected output, {{< keyseq >}}C-x
C-b test RET{{< /keyseq >}}:

```
username=evanmoses
protocol=https
host=testthis.com
password=verysecret
```

We can run our function with `eval-expression`, {{< keyseq >}}M-:
(auth-source-git-credential-helper--process-output) RET{{< /keyseq >}}, and we
should see the result: `(:user "evanmoses" :host "https://testthis.com" :secret
(lambda nil (auth-source--deobfuscate v)))`.  We can see if we can get our
secret back out by doing {{< keyseq >}}M-: (funcall (plist-get
(auth-source-git-credential-helper--process-output) :secret)){{< /keyseq >}}
(that is, get the value of `:secret` from the plist and call it as a function),
and we should see the result is "verysecret".  It worked!

## Wrapping it up
All we have to do is to add our new auth source as a place to look for auth data

```elisp
  (add-to-list 'auth-sources "git-credential-helper")
```

And now `forge` is happily retreiving tokens from our internal tool, and I'm back to writing my PR descriptions in Emacs.  Yak shaved!

You can see the full code in [my dotfiles
repo](https://github.com/emoses/dotfiles/blob/master/emacs/configs/okta.el#L113).
I may try to turn this into a MELPA package, but I think it would have to be
generalized a bit, and I've also never built a MELPA package before, so that's a
whole 'nother yak.
