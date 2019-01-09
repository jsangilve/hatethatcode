Title: When prettier-eslint and Vim don't work as expected
Date:2017-07-06
Tags: vim, javascript, prettier, eslint
Category: javascript
Authors: José San Gil

I think almost everybody agrees that Prettier (Javascript formatter) is a fantastic tool and using it along with Vim is simple. 

There are a couple of Vim plugins for prettier that seem to work well. However, I like to keep Vim as light/fast as possible, so I chose [prettier-eslint-cli](https://github.com/prettier/prettier-eslint-cli) which doesn't require to install a plugin, but add a couple of lines to your Vim's config file (it also combines with [eslint rules](https://github.com/prettier/prettier-eslint)).


So, I added the following two lines to my `.vimrc` after installing prettier-eslint-cli via yarn, and... it didn't work (because computers...).

```
 " Prettier-eslint Javascript formatter
 autocmd FileType javascript set formatprg=prettier-eslint\ --stdin\ --no-semi\ --single-quote
 autocmd BufWritePre *.js :normal gggqG
```

The first line sets `formatprg` (format program) to `prettier-eslint`, i.e., use prettier-eslint as text formatter when the filetype is javascript.  I double checked that `formatprg` was properly set when a Javascript file was open, but it didn't work. After reading different blog posts, I found this in `formatprg` docs (`help formatprg`)

```
 If the 'formatexpr' option is not empty, it will be used instead.
 Otherwise, if 'formatprg' option is an empty string, the internal
 format function will be used |C-indenting|.
```

My `formatexpr` wasn't empty because I had vim-es6 plugin installed (syntax highlighting for ES6+). This plugin set's `formatexpr` for indentation; **I removed vim-es6 and prettier-eslint-cli started to work correctly**.

