---
title:  Emacs 开发环境
classes: wide
categories:
  - 2021-02
tags:
  - emacs
---

在 Emacs 下开发会用到的一些插件和小技巧。

## Python Docker 开发

参照 [https://emacs.stackexchange.com/questions/19431/tramp-method-for-docker](https://emacs.stackexchange.com/questions/19431/tramp-method-for-docker)， 修改 `.emacs`  TRAMP 相关配置：

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

透明修改容器内 python 模块，保存后自动热加载，近似本地开发。


## 微信小程序开发

配合 Win10 自带 Linux 子系统，实现在 linux 端开发小程序应用。

安装 Kali 子系统后，使用 rsync 命令同步 linux 端源码：

```
xiez@BJCYNB00045:~$ while true; do rsync -avz --delete --exclude '.#*' xiez@10.6.0.118:~/dev/oms/frontend/wxapp /mnt/c/Users/A02382/oms/; sleep 4; done
```
