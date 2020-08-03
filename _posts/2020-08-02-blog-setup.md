---
title: "Getting Started, A Blog"
author: Nick Harris
date: 2020-08-02 21:30:00 -0500
tags: [jekyll, github pages, github actions]
description:
image: /assets/img/blog-setup/library.jpg
pin: true
---

# Build, Purchase, Generate, or Borrow

When deciding to setup a personal site I knew I had a few options. Do I build it myself and demonstrate my web designing skills at the same time? Do I purchase a solution? Do I publish my content on some site generator? Or do I borrow from the OSS community? For me, this site was not about demonstrating my web development skills. I know enough CSS to get by and I'm not trying to claim otherwise. I don't have a lot of free time between work and parenting young twins so I would rather that time spent on generating content. So I decided to borrow a free theme that I liked, [Chirpy](https://chirpy.cotes.info/), and use [Jekyll](https://jekyllrb.com) to transform Markdown to HTML for me. This route ended up being more time consuming than I thought despite my background in web development but I'm fairly pleased with the results and the awesome work put into Jekyll and the Chirpy theme. Thank you OSS community! As awesome as these tools are there were some gaps and gotchas with getting it all setup properly so I wanted to document what I had to do to fill in the gaps.

# Jekyll Themes and Filling in the Gaps

Jekyll and the Chirpy theme pretty much work out of the box. After following the well documented theme and jekyll documentation, you can get it running locally pretty quickly. I don't think anyone needs to know much web development to get going with this theme. However, there was a pretty glaring nice to have feature missing, article reading time. This is some nice polish to add to you article when sharing it. I personally really like knowing what I'm getting into before I start so I can mentally prepare for a long read. A more critical missing feature was social media link preview meta tags. Slack calls this unfurling and this is also referred to as Twitter Cards. Searching for help on this pretty challenging to nail down the right keywords. Once you do, it is pretty easy to setup and something I wish Chirpy already had built-in given how feature rich the theme already was. (I definitely owe a PR to contribute back to Chirpy on this one)

## Reading Time

To get this setup I originally went with a very basic word count divided by 5. This ended up seeming very off in the estimation. I then found a [Jekyll reading time plugin](https://github.com/bdesham/reading_time) that was popular and so I ended up going this route. OSS community again to the rescue! Installation and usage was pretty straight forward. I opted to add it to both the [posts.html](https://github.com/nikordaris/nikordaris.github.io/blob/develop/_layouts/post.html#L35-L42) and [home.html](https://github.com/nikordaris/nikordaris.github.io/blob/develop/_layouts/home.html#L76-L84) `_layouts`. The problem was, I was using Github Pages to deploy my Jekyll site magically for me and now my third party plugin broke it...jump ahead to [Third Party Plugins and Github Pages](#third-party-plugins-and-github-pages) section for how to use Github Actions to deploy it for you instead.

## Unfurling for Twitter, Facebook, Slack and LinkedIn

# Third Party Plugins and Github Pages

# Conclusion
