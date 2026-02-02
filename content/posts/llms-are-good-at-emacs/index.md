+++
title = 'Yak Power-Shears: LLMs are pretty good at Emacs'
date = 2026-01-27
categories = ['blog']
tags = ['Emacs', 'llms', 'ai']
summary = """LLMs are good at customizing Emacs.  They can take a minor annoyance that was just one yak-shave too
many and take care of it for you, teaching you some more functionality on the way"""
+++

I've been an Emacs user for more than 25 years, and I'd say I've got a reasonable handle on customizing it. I can write
elisp, and I've got a couple decades of accumulated tweaking cruft on display in [my dotfiles
repo](https://github.com/emoses/dotfiles/tree/master/emacs).  I've written
[advice](https://www.gnu.org/software/emacs/manual/html_node/elisp/Advising-Functions.html), I bind custom functions to keys, and
I'm actually writing this right now with a [customized mode for Hugo
markdown](https://github.com/emoses/dotfiles/blob/master/emacs/configs/hugo-markdown-mode.el) that I ginned up when I
was getting annoyed by the way `fill-paragraph` worked inside Hugo shortcodes.

The reason Emacs is so attractive to those of us who use it is its infinite customizibility.  There's really not much
you can't alter or intercept or rewrite with enough effort, and over time your editor will come to fit you like a
well-worn boot.

However, there are plenty of minor annoyances that I just live with. No one's built the exact package that I want, and
learning enough to do it myself would be [shaving just one
yak](https://projects.csail.mit.edu/gsb/old-archive/gsb-archive/gsb2000-02-11.html) too many.  So I do things by hand,
or use the mouse instead of having a shortcut that jumps to whatever-it-is, or I deal with the hard-to-read log output
instead of nice syntax highlighting.

## LLMs: yak power-shears

LLMs, it turns out, are pretty good at elisp.  There is a ton of open-source training data and good
documentation. So far I've successfully had Gemini or Claude build me:

* A tree-sitter grammar and major mode for the [Cedar policy language](https://docs.cedarpolicy.com/)
* A function to extract backtraces from a json-formatted log that my application at work emits, and pretty-print them
* Syntax highlighting for my application's Go test logs that plays nice with `go-test-mode`.

Unfortunately my biggest successes have all been for work, done on work machines with AI paid for by work, so I can't
publish them here (at some point I'll go through channels and see if I can open source some of it), but here's some
redacted logs that show the pretty formatting I was able to achieve:

![A screenshot of my highlighted logs, with the logs heavily redacted, showing the timestamp, log level, log string, and
json data in different colors and sizes](./log-redacted.png)

And here's a slightly-mangled version of the code that produces it.  Note most of the comments and doc comes straight
from Gemini:

```elisp
(defgroup work-log-faces nil "Faces for Work integration test logs.")

;; Define faces
(defface work-log-timestamp
  '((t :inherit font-lock-comment-face :foreground "#6c757d"))
  "Face for timestamps." :group 'work-log-faces)

(defface work-log-level
  '((t :inherit font-lock-keyword-face :weight bold))
  "Face for log levels (INFO, WARN)." :group 'work-log-faces)

(defface work-log-message
  '((t :inherit font-lock-string-face :foreground "#e0e0e0"))
  "Face for the log message text." :group 'work-log-faces)

(defface work-log-json
  '((t :inherit font-lock-constant-face :height 0.9))
  "Face for the JSON payload." :group 'work-log-faces)

;; Define the matching rules
(defvar work-log-font-lock-keywords
  '(("^\\([0-9T:.-]+\\)[ \t]+\\([A-Z]+\\)[ \t]+\\(.*?\\)[ \t]+\\({.*}\\)?$"
     (1 'work-log-timestamp t)
     (2 'work-log-level)
     (3 'work-log-message)
     (4 'work-log-json)))
  "Regex keywords to highlight structured logs.")

;; 1. Define the control flag (default to nil)
(defvar -work-test-highlighting-active nil
  "If non-nil, apply custom log highlighting in go-test-mode.")

;; 2. Define the function that checks the flag
(defun my:work-apply-logs-highlight ()
  "Apply highlighting only if the Work test flag is active."
  (when -work-test-highlighting-active
    (font-lock-add-keywords nil work-log-font-lock-keywords t)))

;; 3. Add it to the hook
;; Note: go-test usually runs in `go-test-mode` or `compilation-mode`
(add-hook 'go-test-mode-hook #'my:work-apply-logs-highlight)
```

### Specifics

So far I've used two primary interfaces to LLMs: Claude Code, especially [Claude-code-ide for
Emacs](https://github.com/manzaltu/claude-code-ide.el), and the Gemini web interface.  For Claude I've been using Opus
4.5, for Gemini 3 either Thinking or Pro.  Keep in mind that most of the time my company is paying for tokens, so I'm
not being particularly cost-conscious (sorry, bean-counters).  I haven't done any serious comparisons of models and
interface because...all the modern models work pretty well for this sort of thing.

For both syntax highlighting and the backtraces, I was able to simply paste in a sample of the input and describe what
I'd like, and it was able to give me some elisp I could drop into my .emacs.d.

The ts grammar and mode needed a bit more back-and-forth and some actual debugging (by both myself and Gemini).  I think
if I had less elisp experience I wouldn't have been able to vibecode the whole solution, but I've never really written a
ts grammar or a major mode before that was more than a tweak or two, and it got me 80% of what I needed.

## Go forth and customize

The code that Gemini wrote for the font-locking wasn't revolutionary.  It ended up just writing a regex, some faces, and
calling `font-lock-add-keywords`.  But I've barely messed with
font-lock or face definitions myself before and I didn't know any of the relevant functions off the top of my head.  I
could have figured it out, but it wouldn't have been worth my time.  The LLM shaved most of the yak for me.


So next time you're annoyed by a missing motion command, or ugly output, or you wish you had a command that worked *just
a bit* differently, ask Claude/Gemini/your favorite LLM to fix it for you. Not only will you fix your problem, but if
you take just a bit of time to review the output (and you're not gonna just blindly throw LLM code in your .emacs right?
RIGHT?), you may learn about a subsystem you didn't know before. I'm sure you can do this with Neovim or, to a lesser
extent, VSCode or other editors that have reasonable extension points, so go fix them too. Your yaks will thank you come summer.
