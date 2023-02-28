---
title: "Twitch Settings File"
date: 2023-01-21T02:33:15-05:00
toc: true
---

## Background
I do a fair amount of web-based reverse engineering. Mainly on rest APIs, but exploring the files a website sends back is always fun to see what kind of "hidden" data they return. I say hidden, but realistically it's all public. Nothing is stopping you from obtaining a copy of it yourself or even viewing the raw contents. You just have to know where to look and know what you are looking at. In the case of this blog post, a settings file for Twitch. This file was "found" (I say found loosely here because again it's a publicly available file) by a user in a Twitch Partner and then relayed into another Discord I am a part of. The contents of this file intrigued me, so I decided to analyze it and see what it contained.

## How to access the file
Accessing the file directly is quite easy. At the time of writing this, you can go here: [https://static.twitchcdn.net/config/settings.4d4b18dbf1731fa24e3ab14ab82d6992.js](https://static.twitchcdn.net/config/settings.4d4b18dbf1731fa24e3ab14ab82d6992.js) to view it, but due to the abundance of seemingly random characters I'd take a guess this file is dynamically generated and thus its name will change.

So let's talk about how you can "find" it. If you are a web developer, you'll probably find this easy. If not, well, prepare to become one. Start by pressing f12 in a new browser tab. This will open up the `DevTools`. `DevTools` are a core part of being a web developer as it allows you to see a range of helpful information from the webpage console to the webpage source code and beyond. In our case, all we care about is the source code, specifically the files that make up the source code, including files used to store settings and configurations. To access it we must first navigate to [https://twitch.tv/](https://twitch.tv/). Remember, navigate to Twitch in the same tab that you have your `DevTools` open and don't close them. Once Twitch is loaded take a look at the `DevTools` and file the `Sources` tab towards the top of the `DevTools` window and click on it. Now on the left side of the window look for a little window icon with the text `top` next to it. If it's not already expanded to show its contents underneath, expand it now. from there find `static.twitchcdn.net` and expand it. Then find `config` and expand it. In that expanded folder should be the settings file! If you click on it, it should open the contents of the file in the pane to the right. Congrats you are now a certified web dev! Ok, well maybe not, but still, we've got what we came for.

## Analyzing the file
Usually, when you go to analyze a file you have something you are looking for or some sort of starting point. If you don't well, looking at the file as a whole and understanding what it is is a good start, but doesn't really mean much if you can't relate or link it to anything. In this case, my starting data was the name of a streamer I know `Matrixis` whose name is actually in this file, which is what kicked off the initial interest in the file. Throwing the file contents into a JSON prettifier and using `control+f` to find the instance of his name yields this bit of JSON

```json
...
"heartbreak_allowed": [
"tehmorag",
...
"matrixis"
],
...
```

I've removed most of the names in this list just to save space, but in total there are 46 streamers in this list including Matrixis, but what exactly is this list for? Well the list has the name `heartbreak_allowed` which does sound rather ominous, but if we again search this file for `heartbreak_allowed` we can see it is also used in 1 other place in this file

```json
...
"13d9c799-61b8-45ad-bed9-7a9822483576": {
    "name": "memberships_heartbreak_allowed",
    "v": 18346,
    "t": 2,
    "groups": [
        {
            "weight": 0,
            "value": "control"
        },
        {
            "weight": 100,
            "value": "active"
        }
    ]
}
...
```

If we further note the parent element to this bit of JSON we can see it is within a bigger object called `experiments`. In total this `experiments` object has 449 objects within it. That's a lot of experiments! Taking all this and the contents of the object containing `memberships_heartbreak_allowed` we can start to form a guess about what this is all about. Given the parent tag of `experiments` and most objects within it having `groups` that have a group named `control`, this makes me think this is used for site-wide feature testing. Usually, `control` means no change, i.e for these `experiments`, user experiences without said feature enabled. Having a `weight` on each group also makes me think that this is being used to control how many users are put into each group. Most groups have a total weight of 100, but that isn't necessary. For this group, the `active` group has a weight of 100 and `control` has a weight of 0, so I'd guess that this "experiment" has concluded and has been rolled out to all users. I'm not sure if that means all users on that sub-list, or all users on Twitch as I'm not familiar with this feature, but this still seems a solid theory.

### The "Experiments"

If we back up a bit and look at the `experiments` object as a whole we can see that there are a lot of experiments. 449 to be precise. I've extracted the data to a slightly more readable format [here](https://gist.github.com/TheTurkeyDev/81dbeeef519ed812a8e3d4a292d16ac7), but realistically there's still too much going on here. If I narrow this list down to only the experiments that don't have all the weight in 1 group we get a [slightly more manageable list](https://gist.github.com/TheTurkeyDev/c35aaefa05afe829b1e5493d7ba4df05) of only 86 experiments. The names can be a bit vague, but here is a short list of ones I find interesting. I may add some notes as to why later.

```
- gifting_bundle_animations_experiment
- one_click_recurring
- disco_sprig_logged_out
- community-gifting-reduce-friction
- cecg_stream_summary_email
- CSI_LOOK_WHAT_YOU_MADE_ME_DO
- sub_mgmt_modal
- sunlight_streaming_software
- fring_delayed_preroll
- contigo_ojos_y_duende_proxima
- wysiwyg_chat_input
- ad_countdown_timers
- cheering_web_ux_improvements
- subscriber_recap
```

### Rest of the file
Backing away from the experiments and looking at the rest of the file, it's hard to determine exactly what it's doing or is used for. It seems to relate to the below experiment features and site features as a whole, but what exactly 1 and false mean in 
```json
...
"2fa_remember_me": [
    1,
    false
],
...
```
beats me. We can see this file has multi-environment support as it has 
```json
{
"environment": "production",
...
```
to either indicate that this config is for prod, or that when the site accesses this value it then knows it's in prod. I next noticed this section is all about ads.
```json
...
"amazon_ads_url": "https://s.amazon-adsystem.com/iui3?d=3p-hbg&ex-src=twitch.tv&ex-hargs=v%3D1.0%3Bc%3D8858214122683%3Bp%3De75425fb-5407-7bd5-fd20-f462e98a8777",
"amazon_ads_url_crown_uk": "https://aax-eu.amazon-adsystem.com/s/iui3?d=forester-did&ex-fargs=%3Fid%3Db8b26227-de81-5bfb-4046-b9158f6a8c08%26type%3D4%26m%3D3&ex-fch=416613&ex-src=https://www.twitch.tv&ex-hargs=v%3D1.0%3Bc%3D3815840130302%3Bp%3DB8B26227-DE81-5BFB-4046-B9158F6A8C08",
"amazon_ads_url_crown_us": "https://s.amazon-adsystem.com/iui3?d=forester-did&ex-fargs=%3Fid%3D2d452222-ea0d-0b73-d0cc-472923e63141%26type%3D4%26m%3D1&ex-fch=416613&ex-src=https://www.twitch.tv&ex-hargs=v%3D1.0%3Bc%3D7416603020101%3Bp%3D2D452222-EA0D-0B73-D0CC-472923E63141",
"amazon_ads_url_prime_page_uk": "https://aax-eu.amazon-adsystem.com/s/iui3?d=forester-did&ex-fargs=%3Fid%3D5b59d365-e0b2-268d-00a0-aa0f59cce0c1%26type%3D4%26m%3D3&ex-fch=416613&ex-src=https://www.twitch.tv/&ex-hargs=v%3D1.0%3Bc%3D3815840130302%3Bp%3D5B59D365-E0B2-268D-00A0-AA0F59CCE0C1",
"amazon_ads_url_prime_page_us": "https://s.amazon-adsystem.com/iui3?d=forester-did&ex-fargs=%3Fid%3D573a4bd9-f106-f600-a392-699ceaddb160%26type%3D6%26m%3D1&ex-fch=416613&ex-src=https://www.twitch.tv/prime&ex-hargs=v%3D1.0%3Bc%3D7416603020101%3Bp%3D573A4BD9-F106-F600-A392-699CEADDB160",
"amazon_advertising_pixel": "https://s.amazon-adsystem.com/iu3?pid=49226e71-48b6-4ccb-bf4c-f82acb404220",
...
```
It seems this is used to tell the site where to find the ad information, but that's a bit of a shot in the dark. It could be more or less than that. Next up I came across some sections that had more usernames, or user ids. First
```json
...
"ad_content_metadata_allowlist": [
    "155668964",    //qtbros (staff)
    "38543271",     //uberdabing (staff)
    "25100025",     //sticejf (staff & affiliate)
    "24199752",     //themenegg (staff)
    "472905796",    //tropical_penguins (staff)
    "452977866"     //cubicdolphin (staff)
],
...
```
Not quite sure what `ad_content_metadata_allowlist` specifically means here though. Next, I found
```json
...
"badge_flair_overrides": [
    "436929429",    //ccm_test (affiliate)
    "58682589",     //delaxia (staff & affiliate)
    "467487002",    //qa_1zael0110_partner (partner)
    "467832116",    //qa_1zael0110_affiliate (affiliate)
    "28337972",     //limealicious (partner)
    "409749393",    //qa_kevihan_partner (partner)
    "193141706",    //kevihan (affiliate)
    "509785842"     //melody (affiliate)
],
...
```
Obviously this list is some sort of ability to change or override their badge flair (The bits added onto a badge like tier 2&3 subs have), but what I find interesting is `qa_1zael0110_partner`, `qa_1zael0110_affiliate`, and `qa_kevihan_partner`. These seem like some QA accounts have gone rogue and slipped into production. Or maybe they were meant to be here all along?

Not sure what this one is used for, but thought I'd still throw it in
```json
...
"cf_terms_allowlist": [
    "197886470",    //twitchrivals (partner)
    "94753024",     //mizkif (partner) 
    "25385874"      //majorgiggles (staff)
],
...
```

What on earth is the colosseum?
```json
...
"colosseum": [
    "qa_bits_partner",
    "116076154",            //qa_bits_partner (staff & partner)
    "johnnybanana5",
    "529270304",            //johnnybanana5
    "seph",
    "108707191",            //seph (partner)
    "qa_slaye_affiliate",
    "265943326"             //qa_slaye_affiliate (affiliate)
],
...
```
Interesting here is that things are duplicated.

Now this one is interesting. A Copyright Complaint Form can mean a few different things. Are these users that can submit for a copyright notice or are these users who can counter copyright complaints made against them? Likely a testing feature, but interesting non the less.
```json
...
"copyright_complaint_form_user_allowlist": [
  "518822316", //mightymaven (staff)
  "514236910", //chubsywubsyyy  (staff & affiliate)
  "490177374", //wuaiwuai2
  "514820819", //ishneetkaur (staff)
  "191943869", //cfort9 (affiliate)
  "554342166", //smatty_d_mtg (staff)
  "134901385", //emmett_tan (staff)
  "225435142", //vivaciousali (staff & affiliate)
],
...
```

I also noticed a few survey links sprinkled about like so
```json
...
"c2_cel_ux_exp_survey_link": "https://twitchtv.az1.qualtrics.com/jfe/form/SV_6g983xsfyJoU4Zg",
...
```
Nothing too special here, but thought I'd point it out.


This bit also looks interesting:
```json
...
"cc_v2_pov_selector": [
    "overwatchleague",
    "overwatchleague_ru",
    "overwatchleague_fr",
    "overwatchleague_br",
    "overwatchleague_kr",
    "maybe_ill_be_tracer"
],
"cc_v2_single_stream": [
    "overwatchleague",
    "overwatchleague_ru",
    "overwatchleague_fr",
    "overwatchleague_br",
    "overwatchleague_kr",
    "maybe_ill_be_tracer"
],
"cc_v2_whitelist": [
    "cpt_meticulous_test-staff",
    "faceittv",
...
```
Not sure what cc_v2 is, but interesting this seems to be mainly esports based. I wonder if it's some way of linking streams together for larger events?


```json
"celebi_animation_settings": "{}",
"celebi_beta_channels": [
    "seph"
],
"celebi_celebration_area": "EVERYWHERE",
"celebi_stream_delay": true,
```
Hmmmmmmmmmm


There are plenty more things in this file to look at, but right now it's late and I need to stop. This is hardly a comprehensive blog, just some things I found interesting. Reach out if you notice anything or if you have anything to add to what I've said here!



