.gitconfig
```gitconfig
[user]
    name = xingle
    email = xingle@yeptelecom.com
[color]
    ui = true
[core]
    editor = vim
    excludesfile = ~/.gitexcludes
[commit]
    template = ~/.gittemplate
[alias]
    cm = commit -a
    co = checkout
    st = status
    pl = pull
    lg = log
    sh = show
    df = diff
    sa = stash
    br = branch
    cp = cherry-pick
```

.gitexcludes
```gitexcludes
*.com
*.dll
*.exe
*.o

*.log
*.sqlite

ehthumbs.db
Thumbs.db
```
.gittemplate
```gittemplate
xingle: add file
```
