---
title:  Emacs 开发环境
classes: wide
last_modified_at: 2024-04-12T11:20:02+08:00
categories:
  - 2019-10
tags:
  - emacs
---

在 Emacs 下开发已经十多年了，总结下个人觉得能提高工作效率的的一些插件和技巧。

## 键位和快捷键

Caps 和 Ctrl 键互换能非常有效的提高使用 Emacs 甚至系统其他软件的体验。经过几周的习惯后，左下角的 Ctrl 按键基本就不会再触碰，能有效减少手指扭曲和肌肉不适的风险。

使用 Ctrl-x Ctrl-m 来替换默认的 M-x 也能很大的提升 Emacs 的使用体验。

```
;;;M-x
(global-set-key "\C-x\C-m" 'execute-extended-command)
(global-set-key "\C-c\C-m" 'execute-extended-command)
```

## TRAMP for Docker

开发环境容器化已经成为我的标准开发方式，日常工作中，如何快速查看/修改容器里的文件对开发效率有很大的影响。最高效的方式是像本地文件一样访问容器里的文件。

[TRAMP mode](https://www.emacswiki.org/emacs/TrampMode) 可以用来透明访问远程系统的文件，再结合[这里提到的方法](https://emacs.stackexchange.com/questions/19431/tramp-method-for-docker) 就可以达成上面的目标。

修改 TRAMP 相关配置：

```
;; Open files in Docker containers like so: /docker:drunk_bardeen:/etc/passwd
(push
 (cons
  "docker"
  '((tramp-login-program "docker")
    (tramp-login-args (("exec" "-it") ("%h") ("/bin/bash")))
    (tramp-remote-shell "/bin/sh")
    (tramp-remote-shell-args ("-i") ("-c"))))
 tramp-methods)

(defadvice tramp-completion-handle-file-name-all-completions
  (around dotemacs-completion-docker activate)
  "(tramp-completion-handle-file-name-all-completions \"\" \"/docker:\" returns
    a list of active Docker container names, followed by colons."
  (if (equal (ad-get-arg 1) "/docker:")
      (let* ((dockernames-raw (shell-command-to-string "docker ps | perl -we 'use strict; $_ = <>; m/^(.*)NAMES/ or die; my $offset = length($1); while(<>) {substr($_, 0, $offset, q()); chomp; for(split m/\\W+/) {print qq($_:\n)} }'"))
             (dockernames (cl-remove-if-not
                           #'(lambda (dockerline) (string-match ":$" dockerline))
                           (split-string dockernames-raw "\n"))))
        (setq ad-return-value dockernames))
    ad-do-it))
```

保存生效后，就可以像本地文件一样访问容器里的文件。

## JSON / Python-Dict 转换

日常工作中，经常需要对 JSON 对象和 Python 字典对象进行互转，通过下面两个 elisp 函数就能实现选中区域的互相转换，提高效率。

```
(defun python->json (start end)
  (interactive "r")
  (let ((text (buffer-substring-no-properties start end)))
    (kill-region start end)
    (call-process
     "/usr/local/bin/python3"
     nil
     t
     nil
     "-c"
     "import ast; import json; import sys; x=ast.literal_eval(sys.argv[1]); print(json.dumps(x,indent=4))"
     text)))

(defun json->python (start end)
  (interactive "r")
  (let ((text (buffer-substring-no-properties start end)))
    (kill-region start end)
    (call-process
     "/usr/local/bin/python3"
     nil
     t
     nil
     "-c"
     "import sys,json; from pprint import pprint; pprint(json.loads(sys.argv[1]))"
     text)))
```

## 时间戳转换

日常工作中，经常需要查看时间戳对应的具体时间，通常需要借助网页端的在线工具，比较影响效率。通过这个 elisp 函数，就能实现选中区域的转换。

```
(defun timestamp->dt (start end)
  (interactive "r")
  (let ((text (buffer-substring-no-properties start end)))
    (kill-region start end)
    (call-process
     "/usr/local/bin/python3"
     nil
     t
     nil
     "-c"
     "import sys, datetime; print(str(datetime.datetime.fromtimestamp(int(sys.argv[1])))+' UTC+08')"
     text)))
```

## URL/Unicode encode/decode

日常开发中，经常需要对 URL 或者 Unicode 进行编码/解码，通常需要借助网页端的在线工具，比较影响效率。通过下面几个 elisp 函数，就能实现选中区域的转换。

```
;;; utility commands
(defun func-region (start end func)
  "run a function over the region between START and END in current buffer."
  (let ((text (buffer-substring-no-properties start end)))
    (kill-region start end)
    ;; (text (delete-and-extract-region begin end)))
    (insert (funcall func text))))

(require 'url-util)
(defun url-encode-region (start end)
  "urlencode the region between START and END in current buffer."
  (interactive "r")
  (func-region start end #'url-hexify-string))
(defun url-decode-region (start end)
  "de-urlencode the region between START and END in current buffer."
  (interactive "r")
  (func-region start end #'url-unhex-string ))

;; M-x package-install RET unicode-escape RET
;; https://github.com/kosh04/unicode-escape.el
(require 'unicode-escape)
(defun unicode-encode-region (start end)
  "unicode encode the region between START and END in current buffer."
  (interactive "r")
  (func-region start end #'unicode-escape-region))
(defun unicode-decode-region (start end)
  "unicode decode the region between START and END in current buffer."
  (interactive "r")
  (func-region start end #'unicode-unescape-region))
```

## 微信小程序开发

Linux 环境没有好的微信小程序 IDE，而 Windows 的 IDE 键位布局都不大适应。通过 Win10 自带 Linux 子系统，就可以实现在 Emacs 里编辑源码，Windows 端查看效果。

安装 Kali 子系统后，使用 rsync 命令同步 linux 端源码：

```
xiez@BJCYNB00045:~$ while true; do rsync -avz --delete --exclude '.#*' xiez@10.6.0.118:~/dev/oms/frontend/wxapp /mnt/c/Users/A02382/oms/; sleep 4; done
```
