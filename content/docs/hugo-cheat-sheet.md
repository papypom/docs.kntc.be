---
title: "Hugo cheat sheet"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## Create new page

I'm using the [Hugo Book](https://github.com/alex-shpak/hugo-book) Theme, which means it will render the content of the content in the `content/docs` folder as a book.

So the creation of a new page go as such :
```
hugo new content content/docs/my-new-page.md
```

## Serve locally

Simply use `hugo serve --disableFastRender`. The `disableFastRender` option is there to ensure proper site construction, when you add page, for example.

## Publish website

I'm using [rclone](https://rclone.org/), which is set with a SFTP remote named `docs.kntc.be`. Enter the following commands :

```
hugo --gc --minify
rclone sync --interactive public/ docs.kntc.be:
```

The `--interactive` can be omitted if you simply want to push to prod and erase everything already present. So, as I'm copy-pasting these commands, this becomes :
```
rclone sync public/ docs.kntc.be:
```

## Syntax highlighting

For specific code, such as YAML, I use [syntax highlighting](https://gohugo.io/content-management/syntax-highlighting/) and the [emacs style](https://gohugo.io/quick-reference/syntax-highlighting-styles/).

Thus, for YAML the header is ` ```yml {style=emacs}`. To highligh specific lines, add `hl_lines=[2,"4-7"]` between the braces, and to add line numbering `linenos=inline`.