---
layout: single
title: 
---
[https://www.reddit.com/r/cursor/comments/1i18k7h/developed_a_chrome_extension_with_cursor_which/](https://www.reddit.com/r/cursor/comments/1i18k7h/developed_a_chrome_extension_with_cursor_which/)

**fankaidev** - 2025-01-14T15:28:52.000Z

https://preview.redd.it/zpppufh0azce1.png?width=1280&amp;format=png&amp;auto=webp&amp;s=983c81440b2a348d5750632cd97896403705ee13

    I spent the weekend to write a AI-powered chrome extension, All My Web, which makes creating and managing userscripts extremely easy.
    
    It's available in [chrome store](
    https://chromewebstore.google.com/detail/all-my-web/okgfcgmepmeibnkafdaidhpddhcgpcgp
    ) now. However, due to mv3 restriction, you have to enable developer mode to use it. It's still in very early stage, but I already used it for some interesting tasks, and generally works fine.
    
    It's fully open sourced on [github](
    https://github.com/fankaidev/all-my-web
    ) . Most of the code is written with Cursor, though both claude and deepseek struggled to figure out the mv3 restriction on scripts. 😭
    
    Actually I spent quite some time to build my [cursor rule](
    https://github.com/fankaidev/all-my-web/blob/main/.rules
    ), basically ask AI to split requirements to stories, which in turn be split to tasks, so I could verify what's going on. I think cursor is really exceptional, and makes my life much easier. It also inspired my that now we can leverage AI to do so much, including writing scripts to manage all my web content.


---
> **Only-Set-29** - 2025-01-14T23:40:46.000Z
>
> thank you!

---
> **thepantages** - 2025-01-16T15:11:08.000Z
>
> What is possible with these scripts?  Forgive my noob question, but is the use case to add specific functionality to a web page?

---
>> **fankaidev** - 2025-01-20T14:11:12.000Z
>>
>> yes   
>> there has already been great extensions like violentMonkey. you may find all kinds of userscripts in https://greasyfork.org/en. 
>> 
>> what's different here, is that i want to make the generation of scripts fully automatically, so easier for non-coders to get things done

---
> **fankaidev** - 2025-01-23T01:29:38.000Z
>
> This extension now offers free script generation with LLM. If you don't want to set up your own LLM API, you could simply login with Google and enjoy free generations to try out. At present the quota is 20 generations per month, which should be enough for try some interesting ideas.
> 
>![img](https://preview.redd.it/syucdkr2cnee1.png?width=740&amp;format=png&amp;auto=webp&amp;s=bfbcf2954ad3cf24ea2d18d9aa8a627fcbf06762)
