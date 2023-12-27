# Bypassing Discord marked links filter

Recently a friend of mine showed me a way to use hyperlinks on Discord. Discord has a markdown notation support, so users can enhance their messages with formatting, using bold, itallic, titles, etc. It's also possible to use what Discord calls masked links, which are hyperlinks, just like normal markdown, using the syntax `[text](https://example.com)`. The result is the following:

<p align="center">
  <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/874541f4-6d9b-4c2a-b0e7-08dff04d1b36" />
</p>

In the context of cybersecurity, one of the first things that comes to mind is to try to make a fake link, putting another URL on the "text" part. It would be something like `[https://malicious.com](https://example.com)`. By doing so, an user would see the URL `https://example.com`, but by clicking on it, he would be redirected to `https://malicious.com`.

However, Discord has a filter to prevent it. It's not possible to add an URL on the text part. If you try to do so, the hyperlink is not created and the original raw message is displayed:

<p align="center">
  <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/96ca761b-5d31-45fc-ac03-a7ce0a2fcff6" />
</p>

After some tests, I figured out that Discord was blocking the text part from having a URL containing `http://`. It is then possible to make such link suppressing this part, which leads more or less to the desired results:

<p align="center">
  <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/248d2ac7-2919-4db7-987e-ce0837ca068e" />
</p>

# Trying to bypass the filter

I tried some things to send a link with the `http://` at the beginning; different protocols, different characters (e.g. using the character `╱` as the slash in `https:╱╱`) but it didn't work.
After some analysis of the client-side javascript code of Discord, I found the function that restricts characters such as `╱` (someone from Discord has already thought about it hehe) and was able to find some ways to visually bypass the "https://" restriction.

<p align="center">
  <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/f575670b-d2a1-43c8-b198-345c3ac31f1b" />
</p>

This function restricts a bunch of characters that look like the slash "`/`" and the colon "`:`", but there are some that are not on that list, such as `୵`, `⁏` and `⁚`.

It is then possible to craft some fake links combining those characters:

```
[https⁏//discord.com](<https://malicious.com>)
[https⁚୵୵discord.com](<https://malicious.com>)
[https:୵୵discord.com](<https://malicious.com>)
[https⁏୵୵discord.com](<https://malicious.com>)
```
<p align="center">
  <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/6740ee9f-8394-45d4-876f-3ca3e8d4d8e9" />
</p>

The third one looks pretty convincing, but all of them uses characters that may break depending on what font the user is using.

However, there is another way to make such fake URLs: using bold or itallic. It turns out that if the colon or slash characters are bold or itallic, the filter does not apply, allowing some cool combinations:

```
[https*://*discord.com](<https://malicious.com>)
[https*:*//discord.com](<https://malicious.com>)
[https:*//*discord.com](<https://malicious.com>)
[https:*||*discord.com](<https://malicious.com>)
[https*:*⁄⁄discord.com](<https://malicious.com>)
[**https://**discord.com](<https://malicious.com>)
[**https:**//discord.com](<https://malicious.com>)
[**https://**discord.com](<https://malicious.com>)
[https:***||***discord.com](<https://malicious.com>)
[**https:**⁄⁄discord.com](<https://malicious.com>)
[__https__*__:__*__//discord.com__](<https://malicious.com>)
```

In my opinion, the fourth one (`[https:*||*discord.com](<https://malicious.com>)`) is the one that is most similar to legitimate URLs. 

Note: There might be other characters that resemble the bar and colon characters, enabling further variations of the misleading links.

<p align="center">
  <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/086bed33-16e5-4682-8d50-0b3680c4b2a5" />
</p>

# The whitelist

Still, when the user clicks on the link, Discord shows a pop up saying that the clicked URL is trying to redirect the user to other site, making it more difficult to convince someone.

<p align="center">
  <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/f4a059ed-7eda-49db-ab9e-1a9c33fa414c" />
</p>

This pop up is not shown when you use `discord.com` instead of `malicious.com`, which means there is a domain whitelist.

Analyzing some Discord javascript files, I found the function `isDiscordUrl`, which determines whether a URL is a valid Discord URL, and if so, the pop up is not shown.

This verification is made by checking the user URL against the regex `(?:^|\.)(?:discordapp|discord)\.com$`

<p align="center">
  <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/d81beb5e-daad-494f-885e-055c30eb7854" />
</p>
<p align="center">
  <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/1abec04a-6aed-4603-8c27-aa280372be6a" />
</p>

I also found that all these domains and subdomains are whitelisted, although I couldn't determine where in the code they are defined:

```
*.discord.com
*.discordapp.com
discord.gg
discord.gift
discord.new
```

That means that if there is an open redirect or a XSS in any of these domains, this can be chained with the bypass I just showed to craft very convincing malicious links (e.g. a seemingly legit `https://gmail.com` link that redirects to a phishing page).

I've searched for these vulnerabilities in these domains/subdomains last few months but couldn't find anything useful for this purpuse. 

The closest aspect I found related to my search is that when a user sets up OAuth2 for their bot in Discord's developer application settings, they can specify the URL to which a user will be redirected after consenting to add the bot to their server.

<p align="center">
   <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/3a384ee4-68b8-4081-b4f4-d9337f6fe935"/>
</p>

This generates the following URL, containing the ID of the bot, some permissions and the actual URL that the user will be redirected to:

```
https://discord.com/api/oauth2/authorize?client_id=REDACTED&response_type=code&redirect_uri=https://google.com/&scope=identify
```

But the request that redirects the user to the provided URL is made via POST by the browser only after the user consents to adding the bot to the server; not a GET, and it required the user's token, so unfortunately it cannot be exploited as we need, although it can be a interesting vector for social engineering attacks.

<p align="center">
   <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/c5b6b758-01d6-41d2-a9b9-2987679dcc7f"/>
</p>

# Spooky downloads

Not only that, but since all Discord attachments are stored in `cdn.discordapp.com`, which is a whitelisted subdomain, we can do things like that:

<p align="center">
   <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/3ea64e96-c4f5-4896-a253-68013342dd90" />

</p>

# #FakeNews

Every time you send a message containing an anchored URL (e.g. `https://google.com`, instead of just `google.com`), Discord make a GET request to the URL and tries to read the HTML meta tags to provide the users on the chat some information about the page, such as title, subtitle, author, website and cover image.

Since all those information are get from the HTML, we can upload our own HTML file containing some crafted meta tags to make things like that:

<p align="center">
 <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/5087340c-515e-4afb-9318-9234f1cfc633"/>
</p>

(Obviously Mrs. Clinton didn't say that, and would never say that unless she know something we don't know)

<p align="center">
 <img src="https://github.com/Nickguitar/bypassing_discord_masked_links_filter/assets/3837916/4a658156-4e75-4458-8124-764e28fcccc9"/>
</p>

# Disclosure and Discord's response

After I identified the issue, I sent a report to Discord on their HackerOne bug bounty program, stating about the risks, that by clicking on a malicious link, an user could have some information leaked, such as IP Address, User Agent, OS type and version, GPU info, etc. affecting directly their confidentiality, and also that it is possible that the link forces the user to download malicious software, or redirects them to phishing pages, affecting directly their integrity. It is also possible to poison the meta tags of a HTML page and send messages containing very convincing fake tweets, fake news, etc.

This was their answer, just a couple hours after I submited the report:

```
Thank you for this report! After some internal discussion it looks like this functionality still 
hits the pop up modal which warns the end user about the real URL (malicious.com in these examples) 
correctly. Combined with this inherently being a phishing issue (which is marked as out of scope on 
our bug bounty program) and suspicious link warnings being best effort mitigation on our bug bounty 
program we are going to close this as a won't fix.
```
