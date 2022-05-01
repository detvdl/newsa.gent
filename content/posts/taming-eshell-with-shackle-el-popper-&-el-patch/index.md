+++
title = "Taming eshell with shackle.el, popper & el-patch"
author = ["Detlev Vandaele"]
date = 2022-05-01T16:37:00+02:00
lastmod = 2022-05-01T18:36:29+02:00
categories = ["emacs"]
draft = false
+++

A short while ago, I decided to adopt eshell into my daily work routine as a quick &amp; easy-to-access shell from within my favourite editor. <br/>
But I couldn't stand some of its default behaviour. Let's talk about it! <br/>

<!--more-->


## Emacs window management {#emacs-window-management}

For quite some time now, I had been unhappy with Emacs' default management of temporary/ephemeral buffers and windows. <br/>

**Example**: Something goes wrong, a `*Backtrace*` buffer pops up in some window, you close it with `q` and **boom**, your window setup is broken yet again. <br/>

That's a very tame and relatable example, but there are some packages that have much more... _intrusive_ behaviour. <br/>

Eshell, for example, just takes over whatever window you are currently in.  That's not very friendly, nor is it preferable for an ephemeral shell, which you don't **always** want on-screen. <br/>

Looking into packages to manage this type of buffer and window more carefully lead me to the [popper](https://github.com/karthink/popper) package. <br/>


## Introducing Popper.el {#introducing-popper-dot-el}

Popper.el is an amazing package, with a fairly simple premise: <br/>

> If it tends to pop up out of nowhere, treat it as a pop-up buffer. <br/>

These "ephemeral" buffers get a window of their own, whose visibility can be toggled on-and-off at your command. <br/>

Its setup is simple enough, and in my `init.el` I have the following configured: <br/>

```emacs-lisp { hl_lines=["8-34"] }
(use-package popper
  :straight t
  :after shackle
  :bind (("C-`" . popper-toggle-latest)
         ("M-`" . popper-cycle)
         ("C-M-`" . popper-toggle-type))
  :config
  (setq popper-group-function #'popper-group-by-projectile
        popper-display-control nil
        popper-reference-buffers
        '(occur-mode
          grep-mode
          locate-mode
          embark-collect-mode
          deadgrep-mode
          "^\\*deadgrep"
          help-mode
          compilation-mode
          ("^\\*Compile-Log\\*$" . hide)
          backtrace-mode
          "^\\*Backtrace\\*"
          "^\\*eshell"
          ("^\\*Warnings\\*$" . hide)
          "^\\*Messages\\*$"
          "^\\*Apropos"
          "^\\*eldoc\\*"
          "^\\*TeX errors\\*"
          "^\\*ielm\\*"
          "^\\*TeX Help\\*"
          "\\*Shell Command Output\\*"
          ("\\*Async Shell Command\\*" . hide)
          "\\*Completions\\*"
          "[Oo]utput\\*$"
          "^magit*"))
  (popper-mode +1)
  (popper-echo-mode +1))
```

As you can tell from the highlighted lines, you **do** need to configure for yourself which buffers are treated as pop-ups. <br/>
But that's a plus, since popper does not make any assumptions about **what** defines a pop-up buffer. That's all up to you! <br/>

The special buffers now get a nice visible `POP` identifier in the leftmost corner of the modeline; it looks sort of like this: <br/>

{{< figure src="/images/Posts/2022-05-01_14-55-14_Screenshot 2022-05-01 at 14.54.19.png" >}} <br/>

As stated in the [popper README](https://github.com/karthink/popper#managing-popup-placement) itself, however, you will need additional configuration to manage the **placement** of these popups. <br/>

By default, it uses, well, the default window placement defined by the creator of the buffer, and if that's non-existent, it pops up at the bottom of your active frame. <br/>

You can set a different default behaviour for popper by setting <br/>
`(setq popper-display-control t)` and defining your own placement function similar to how `popper-select-popup-at-bottom` is defined. <br/>

For more granular control over each window's placement, they recommend [shackle.el](https://depp.brause.cc/shackle/), and so that is what I ended up using. <br/>


## Shackling your windows in-place {#shackling-your-windows-in-place}

Shackle is very similar to popper in configuration, in that it is <br/>

1.  Very simple and straightforward, and <br/>
2.  You just specify a list of regexps, buffer names or modes to match <br/>

and it does its magic. Just see for yourself: <br/>

```emacs-lisp
(use-package shackle
  :straight t
  :demand t
  :config
  (setq shackle-default-rule '(:select t)
        shackle-rules
        '(;; Below
          (compilation-mode
           :noselect t :align below :size 0.33)
          ("*Buffer List*"
           :select t :align below :size 0.33)
          ("*Async Shell Command*"
           :noselect t :align below :size 0.20)
          ("\\(?:[Oo]utput\\)\\*"
           :regexp t :noselect t :align below :size 0.33)
          ("\\*\\(?:Warnings\\|Compile-Log\\|Messages\\|Tex Help\\|TeX errors\\)\\*"
           :regexp t :noselect t :align below :size 0.33)
          (help-mode
           :select t :align below :size 0.33)
          ("*Backtrace*"
           :noselect t :align below :size 0.33)
          (magit-status-mode
           :select t :align below :size 0.66)
          ("magit-*"
           :regexp t :align below :size 0.33)
          ("^\\*deadgrep"
           :regexp t :select t :align below :size 0.33)
          ("^\\*eshell"
           :regexp t :select t :align below :size 0.20)
          ;; Right
          ("\\*Apropos"
           :regexp t :select t :align right :size 0.45)
          )
        )
  (shackle-mode +1))
```

With this combination in hand, I managed to tackle almost every issue I had with buffer- and window-placement. <br/>

**Almost** every single one...except for the main topic of this post: **eshell**. <br/>


## The good, the bad, and the eshell {#the-good-the-bad-and-the-eshell}

As is mentioned in the _Internals_ section of the [shackle.el README](https://depp.brause.cc/shackle/): <br/>

> ... <br/>
> Emacs packages that neither use the display-buffer function directly nor indirectly won't be influenced by shackle. <br/>

And this is problematic for us. If we look at the source code for eshell, we see the following: <br/>

```emacs-lisp { hl_lines=["13"] }
(defun eshell (&optional arg)
  (interactive "P")
  (cl-assert eshell-buffer-name)
  (let ((buf (cond ((numberp arg)
                    (get-buffer-create (format "%s<%d>"
                                               eshell-buffer-name
                                               arg)))
                   (arg
                    (generate-new-buffer eshell-buffer-name))
                   (t
                    (get-buffer-create eshell-buffer-name)))))
    (cl-assert (and buf (buffer-live-p buf)))
    (pop-to-buffer-same-window buf)
    (unless (derived-mode-p 'eshell-mode)
      (eshell-mode))
    buf))
```

That highlighted line, `(pop-to-buffer-same-window buf)` is the bane of our existence at this point. <br/>
No matter what rules you add to `display-buffer-alist`, eshell won't care. It will force its buffer into your current window, regardless of your demands. <br/>

**So how do we fix this?** <br/>
Surely we won't wait for an upstream patch to be applied, especially since the [Emacs 28.1 release notes](https://github.com/emacs-mirror/emacs/blob/5a223c7f2ef4c31abbd46367b6ea83cd19d30aa7/etc/NEWS#L842-L843) state the following: <br/>

> 'project-shell' and 'shell' now use 'pop-to-buffer-same-window'. <br/>
> This is to keep the same behavior as Eshell. <br/>

That's where [el-patch](https://github.com/raxod502/el-patch) comes in! <br/>


## Patch, patch, patch to your heart's content {#patch-patch-patch-to-your-heart-s-content}

Without going too much in-depth, el-patch is a wonderful package once you start wanting to hack on internal packages, or don't want to fork an entire project for a minor code-change. <br/>

Its documentation is extensive, and you can find plenty examples of how to use it in the wild. <br/>

For this post, we'll focus on the following functionalities: <br/>

1.  Use `display-buffer` to manage the eshell-buffer <br/>
2.  Allow specifying a custom buffer-name suffix <br/>

The latter is just to "namespace" our eshell buffers a bit more clearly than just `eshell<1>`, `eshell<2>`, and so on. <br/>

```emacs-lisp { hl_lines=["20-24","28-33"] }
(use-package el-patch)

(eval-when-compile
  (require 'el-patch))

(use-package esh-mode
  :straight (:type built-in)
  :config/el-patch
  (defcustom eshell-buffer-name "*eshell*"
    :type 'string
    :group 'eshell)
  (defun eshell (&optional arg)
    (interactive "P")
    (cl-assert eshell-buffer-name)
    (let ((buf (cond ((numberp arg)
                      (get-buffer-create (format "%s<%d>"
                                                 eshell-buffer-name
                                                 arg)))
                     (arg
                      (generate-new-buffer (el-patch-swap
                                             eshell-buffer-name
                                             (format "%s[%s]"
                                                     eshell-buffer-name
                                                     arg))))
                     (t
                      (get-buffer-create eshell-buffer-name)))))
      (cl-assert (and buf (buffer-live-p buf)))
      (el-patch-swap (pop-to-buffer-same-window buf)
                     (display-buffer buf 'display-buffer-pop-up-window))
      (el-patch-wrap 1 0
        (with-current-buffer buf
          (unless (derived-mode-p 'eshell-mode)
            (eshell-mode))))
      buf))
  :config
  (defun eshell-here ()
    "Opens up a new shell in the directory associated with the
    current buffer's file. The eshell is renamed to match that
    directory to make multiple eshell windows easier."
    (interactive)
    (let* ((parent (if (buffer-file-name)
                       (file-name-directory (buffer-file-name))
                     default-directory))
           (name   (car (last (split-string parent "/" t)))))
      (eshell name)))
  (global-set-key (kbd "C-!") 'eshell-here))
```