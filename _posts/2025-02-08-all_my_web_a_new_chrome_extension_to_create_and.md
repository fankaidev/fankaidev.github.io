---
layout: single
title: 
---
[https://www.reddit.com/r/chrome_extensions/comments/1i1873x/all_my_web_a_new_chrome_extension_to_create_and/](https://www.reddit.com/r/chrome_extensions/comments/1i1873x/all_my_web_a_new_chrome_extension_to_create_and/)

**fankaidev** - 2025-01-14T15:12:28.000Z

I just spent the weekend to write a AI-powered chrome extension, All My Web, which makes creating and managing userscripts extremely easy.

It's available in [chrome store](https://chromewebstore.google.com/detail/all-my-web/okgfcgmepmeibnkafdaidhpddhcgpcgp) now. However, due to mv3 restriction, you have to enable developer mode to use it. It's still in very early stage, but I already used it for some interesting tasks, and generally works fine. 

It's fully open sourced on [github](https://github.com/fankaidev/all-my-web) . Most of the code is written by LLM, though both claude and deepseek struggled to figure out the mv3 restriction on scripts. 😭

Looking forward to your feedback.


---
> **Is_Kub** - 2025-01-14T15:55:59.000Z
>
> Have you checked out tampermonkey? They are the most popular extension right now. You might want to add features they don’t have, to stand out

---
>> **fankaidev** - 2025-01-14T23:43:30.000Z
>>
>> of course there have been great extensions including tampermonkey and violentmonkey, which I myself used quite a lot. But it require programming skills to fully utilize, and it's very hard for normal users to customize their scripts.
>> 
>> I think the biggest difference here is to let AI automate the flow of generating scripts. Actually   , I think users might not need to care about the notion of 'scripts' in the future, they just describe what they want with any website and it's done

---
>>> **Is_Kub** - 2025-01-15T00:12:15.000Z
>>>
>>> Oh I missed that part. Which LLM did you use?

---
>>>> **fankaidev** - 2025-01-15T01:35:52.000Z
>>>>
>>>> at present, actually user choose whatever they want ... have to set up LLM themselves. 
>>>> 
>>>>   
>>>> i might later provide some free trial, maybe use deepseek which is cheap yet capable

---
> **fankaidev** - 2025-01-23T01:30:58.000Z
>
> This extension now offers free script generation with LLM. If you don't want to set up your own LLM API, you could simply login with Google and enjoy free generations to try out. At present the quota is 20 generations per month, which should be enough for try some interesting ideas.
