+++
title = 'Emacs Tidbit: rectangle-number-lines'
date = 2024-04-24
categories = ['blog']
tags = ['emacs']
+++

Emacs has so many little features that do exactly what you need in very specific circumstances.  Here's one that I just
found out about: `rectangle-number-lines`.  I had some lines and I wanted to insert line-numbers for reference.  I
wanted this

```
arguments list: [
team8721038424565681034
user
fdff6f87-d876-4558-8d12-19e039e5a880
a
foo
secret
alsothis
b
e46e2d16-b3ba-4f84-826f-5e3031465e21
]
```

To be this

```
arguments list: [
 1 team8721038424565681034
 2 user
 3 fdff6f87-d876-4558-8d12-19e039e5a880
 4 a
 5 foo
 6 secret
 7 alsothis
 8 b
 9 e46e2d16-b3ba-4f84-826f-5e3031465e21
]
```

I started thinking about kmacros, which I rarely use but have an incrementing counter you can use for this, but I started
googling and lo and behold I found a Stack Overflow post with the answer.  Just highlight a rectangle on the first
column[^1] and type {{< keyseq >}}C-x r N{{< /keyseq >}} which runs `rectangle-number-lines`, which does exactly what I
wanted.

[^1]: Since I use `evil`, I just type {{< keyseq >}}C-v{{< /keyseq >}} to enter visual block mode, but I looked it
    up for vanilla Emacs and {{< keyseq >}}C-x SPC{{< /keyseq >}} to set the rectangle mark.
