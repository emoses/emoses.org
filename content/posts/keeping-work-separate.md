+++
title = 'Separating work and personal config'
date = 2025-04-21
categories = ['blog']
tags = ['emacs', 'git', 'configuration']
+++

{{< tldr >}}
* My dotfiles are checked into a git repo
* I want to avoid checking in sensitive, work-specific config to a public git repo
* I updated my git and emacs configs to check for local overrides in `~/.local`
{{< /tldr >}}

Like many folks, I check my dotfiles into a [git repo](https://github.com/emoses/dotfiles) to share them across multiple machines.  When I set up a new
machine I can simply clone that repo, run an install script that adds some symlinks, and get to work with my happy
personalized config. {{< aside >}} Please don't judge the install.sh too harshly.  My dotfiles repo has accumulated
plenty of cruft over the years, but it pretty much works, and that's valuable. I'm also aware of [GNU
Stow](https://www.gnu.org/software/stow/) and the [Nix-based
home-manager](https://github.com/nix-community/home-manager), but I have a working setup and I haven't felt the need to
shave that particular yak. {{< /aside >}}

My work machine needs to have some special configuration.  I'm required to use certain security tools, there are some
dev tools I only use at work, plus obviously I need to use my work GitHub account.  The security team at my company has
(wisely) asked me to keep the work-specific configuration out of my public GitHub, in order to prevent attackers from
performing reconnaissance on our internal tooling and network configuration.  Here's how I managed that.

## Git config

Our internal git setup uses a credential helper, a tool I've written about at {{< reftitle
"emacs-custom-auth-source" >}}.  In order to use it, we need this in our gitconfig:

```ini
...
[core]
	sshCommand = /usr/local/bin/awesome-security-tool ssh
...
[credential]
	helper = /usr/local/bin/awesome-security-tool helper
```

I added this line near the to of my `dotfiles/git.config` (which is
symlinked to `~/.gitconfig`):

```ini
[include]
    path = ~/.local/git.config
```

and then moved the work-specific config to `~/.local/git.config`.  Git is happy to silently ignore the include if
there's nothing at that path, so I can safely sync this config down to any machine even if it doesn't need any local
configuration.

### Per-repo overrides

On the other hand, when I'm working on non-work repos on my work machine, I want to use my personal configuration
instead. I also want to make sure I'm *not* using the fancy credential helper,
because the credentials it gets won't work with my personal account.

{{< aside >}}I recently learned you can match on the git remote URL in `includeIf` rather than the local directory name, so I may
 update my `includeIfs`.{{< /aside >}}

This can be configured on a per-repo basis in the
`.git/config` for each repo (or by using `git config` without the `--global` flag), but git has a neat feature that will
conditionally include a config file based on the path to the repo.  I check out my personal repos to a particular
directory, so I at this config *at the bottom* of my gitconfig[^1]:



[^1]: When you use `include` or `includeIf`, git treats it as if the contents of the included file were at the point
    where your include directive is.  Since later configs override earlier configs with the same name, if you want
    per-repo overrides you'll need them to be at the bottom or they'll simply be overwritten by the "default" configs.
    More info in the [Git docs](https://git-scm.com/docs/git-config#_includes)

```ini
[includeIf "gitdir:~/dev/github.com/emoses/"]
   path = ~/.gitconfig.personal
```

Where `.gitconfig.personal` looks like

```ini
[user]
        email = webmaster@emoses.org
[core]
        # Override the ssh command from the work-specific config back to standard
        sshCommand = /usr/bin/ssh
```


## Emacs configs

My emacs also has some work-specific customizations, which I keep in a file called `work.el` that used to be checked in
to my dotfiles repo.  My routine for loading all my customizations in my `.emacs` looks like this (if you're really
curious you can find the rest [here](https://github.com/emoses/dotfiles/blob/master/dot.emacs#L297 "A link to the
my:load-config-file function in my dot.emacs")) :

```emacs
(defvar my:osx (eq system-type 'darwin))

(my:load-config-file '("package-bootstrap.el"
		       (lambda () (if my:osx "osx.el" nil))
               "evil.el"
                ;; A bunch more files
                "work.el" ;; Whoops, this was checked in, let's fix that
                ))
```

I added a routine to load any configs from `~/.local/emacs` if they're present.

```emacs
(defconst my:LOCAL_CONFIG_PATH (file-name-concat (getenv "HOME") ".local" "emacs"))
(when (file-exists-p my:LOCAL_CONFIG_PATH)
    (let ((local-el-files (directory-files my:LOCAL_CONFIG_PATH t "\.elc?$")))
      (dolist (local-el local-el-files)
        (load local-el)
        (message "Loaded local config file: %s" local-el))))
```

So I moved my `work.el` out of my dotfiles repo to `~/.local/emacs/work.el` and now I'm all set.
