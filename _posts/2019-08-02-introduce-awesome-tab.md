---
layout: post
title: "Jump Between Buffers in a Slick Way"
date: 2019-08-02
---

Typically, Emacs users have a lot of buffers. People usually use commands like `counsel-ibuffer`, `ivy-switch-buffer`, `helm-buffer-list` to filter buffers by name, and jump to one of them. These are pretty nice methods, but you need to think about the buffer name, and look for the correct one when there are files with the same name. I often find this distracting when I am focusing on my code. It's kind of like filtering the desired buffer out of a bucket of buffers, and "filtering", no matter how good is the mechanism designed, will always give you some mental load:

```
        target buffer
             ↑
         searching
             ↑
|                          |
| awesome.el               |
|   style.css    page.html |
|          *scratch*       |
| blog-post.md             |
|                 ...      |
|  ...        init.el      |
+--------------------------+
```

Another problem is with such a bucket, since buffers are absolutely not organized, we never get around to find unused ones and kill them. Finally, it becomes a sea of buffers, and people invent things like `ibuffer` to deal with it. With `ibuffer` you can mark / save / kill / filter / sort all your buffers, which just give us more mental load. (`ibuffer` itself brings you an `ibuffer` buffer!)

I personally *love* using tabs to organize buffers. I don't know why Emacs users are so against them, maybe tabs just remind us of the "browsing the internet with a mouse" experience, but actually tabs can be very efficient and suitable for keyboard use. Think about it, typically in an Emacs instance we work on 2 or 3 projects, and in each project we may have 3 or 4 frequently used buffers, so buffers can be organized into a matrix:

```
Project 1: | init.el   | awesome.el | ...
           |-----------+------------+--------------+-----
Project 2: | style.css | page.html  | blog-post.md | ...
           |-----------+------------+--------------+-----
Project 3: | Makefile  | header.h   | src.c        | ...

... = not frequently used buffers
```

With such a matrix, we can move to any frequently used buffers with just a few "arrow-key-like commands", and we can easily move to unused ones in current project and kill them. Andy Stewart's [`awesome-tab`](https://github.com/manateelazycat/awesome-tab) gives us exactly the same mechanism, so I would like to introduce it to you.

## Groups and Arrow-Key-Like Commands

awesome-tab uses `awesome-tab-buffer-group` to group buffers. The code is easy to read, basically it divides emacs-related buffers into several group, then uses funtionalities in `project.el` to group remaining buffers according to their belonging projects. You can customize the `awesome-tab-buffer-groups-function` variable to use your own rules.

Only buffers in current group will be shown as tabs. The arrow-key-like commands to browse the buffer matrix are:

- `awesome-tab-backward-tab`
- `awesome-tab-forward-tab`
- `awesome-tab-backward-group`
- `awesome-tab-forward-group`

I've bound them to `;`, `'`, `(` and `)` under my leader key (I swiched `()` and `[]` keys). Look this, I first switch between tabs, then between groups. Think of it as moving in a matrix.

![switching between tabs and groups](/img/introduce-awesome-tab/arrow-key.gif)

## Jump Back and Forth, Make First Screen Nice

What if I want to move to a tab far away? You can use `awesome-tab-ace-jump` (created by me =w=). It's like ace-jump-mode or avy. When calling it, one or two chars appear on each tab, and you can jump to one by typing the chars:

![ace jump to tabs](/img/introduce-awesome-tab/ace-jump.gif)

The default ace strings are made up with `j`, `k`, `d` and `f`, so you can access up to 16 tags within 2 keystokes using index/middle fingers. I don't have a wide screen that can show more than 9 tabs, so I customized `awesome-tab-ace-keys` to be home row letters, thus I can jump with only one key.

What if I want to move to a tab out of current screen? Well, you have to use the searching method (ivy/helm). But the correct way to use tabs is to move frequently used buffers into the first screen, then stay there, so very occasionally we have to search for buffers. Let me show you how to do it.

First you need this function. (Andy has not decided if this is the best way. Hopefully this command will get into awesome-tab's codebase.)

``` emacs-lisp
(defun awesome-tab-move-current-tab-to-beg ()
  "Move current tab to the first position."
  (interactive)
  (let* ((bufset (awesome-tab-current-tabset t))
         (bufs (copy-sequence (awesome-tab-tabs bufset)))
         (current-tab-index
          (cl-position (current-buffer) (mapcar #'car bufs)))
         (current-tab (elt bufs current-tab-index)))
    (setq bufs (delete current-tab bufs))
    (push current-tab bufs)
    (set bufset bufs)
    (awesome-tab-set-template bufset nil)
    (awesome-tab-display-update)))
```

When you move to a distant tab, you can:

- Decide it would be used frequently, so move it to the first position.
- Decide it will not be used frequently, and later go back to the first tab using `awesome-tab-select-beg-tab`.

After several cycles, you have all your frequently used buffers in the first screen, and can easily jump to anyone using `awesome-tab-backward/forward-tab` commands or ace jump. Isn't this slick?

## Bonus: Hydra Makes it Easier

Here's a hydra for you to do consecutive moves in the buffer matrix. It can also move between windows and manage your buffers, I find it way easier to use than `ibuffer`:

``` emacs-lisp
(defhydra toki-fast-switch (:hint nil)
  "
 ^^^^Fast Move             ^^^^^^Tab                        ^^Search            ^^Misc
-^^^^--------------------+-^^^^^^-----------------------+-^^----------------+-^^---------------------------
   ^_k_^   prev group    | _C-a_^^^^       select first | _b_ search buffer | _C-k_   kill buffer
 _h_   _l_  switch tab   | _C-e_^^^^       select last  | _g_ search group  | _C-S-k_ kill others in group
   ^_j_^   next group    | _C-j_^^^^       ace jump     | ^^                | ^^
 ^^0 ~ 9^^ select window | _C-h_/_C-l_/_A_ move to      | ^^                | ^^
-^^^^--------------------+-^^^^^^-----------------------+-^^----------------+-^^---------------------------
"
  ("h" awesome-tab-backward-tab)
  ("j" awesome-tab-forward-group)
  ("k" awesome-tab-backward-group)
  ("l" awesome-tab-forward-tab)
  ("0" my-select-window)
  ("1" my-select-window)
  ("2" my-select-window)
  ("3" my-select-window)
  ("4" my-select-window)
  ("5" my-select-window)
  ("6" my-select-window)
  ("7" my-select-window)
  ("8" my-select-window)
  ("9" my-select-window)
  ("C-a" awesome-tab-select-beg-tab)
  ("C-e" awesome-tab-select-end-tab)
  ("C-j" awesome-tab-ace-jump)
  ("C-h" awesome-tab-move-current-tab-to-left)
  ("C-l" awesome-tab-move-current-tab-to-right)
  ("A" awesome-tab-move-current-tab-to-beg)
  ("b" ivy-switch-buffer)
  ("g" awesome-tab-counsel-switch-group)
  ("C-k" kill-current-buffer)
  ("C-S-k" awesome-tab-kill-other-buffers-in-current-group)
  ("q" nil "quit"))
```

`my-select-window` is a command that automatically recognizes the number in your keystroke and switch to the corresponding window. Below is an implementation using `ace-window`:

``` emacs-lisp
;; winum users can use `winum-select-window-by-number' directly.
(defun my-select-window-by-number (win-id)
  "Use `ace-window' to select the window by using window index.
WIN-ID : Window index."
  (let ((wnd (nth (- win-id 1) (aw-window-list))))
    (if wnd
        (aw-switch-to-window wnd)
      (message "No such window."))))

(defun my-select-window ()
  (interactive)
  (let* ((event last-input-event)
         (key (make-vector 1 event))
         (key-desc (key-description key)))
    (my-select-window-by-number
     (string-to-number (car (nreverse (split-string key-desc "-")))))))
```

When you feel like you need a cleanup, call it, move around windows and buffers, rearranging them, and kill the unused ones. I put too many commands in it, and you should definitely create your own hydra with commands you like to use.
