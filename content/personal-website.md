+++
title = "How I Made This Blog?"
date = "2016-11-20T16:30:00+03:00"
Categories = ["Personal", "Project", "Continous Integration"]
Tags = ["wercker", "github", "github actions", "markdown", "hugo", "golang", "ghpages"]
+++

> After the release of [Github Actions](https://github.com/features/actions) i refactored this project to work with it instead of wercker. Blog posts and configuration for the Hugo is stored under the *source* branch of [Yengas/yengas.github.io](https://github.com/Yengas/yengas.github.io/tree/source), Github Actions builds and pushes html/css/js files on the *master* branch of the same repository.

I haven't been designing any GUI's since i switched from Web development to other platforms. Most of the projects i've worked on for the last few years were either single page designs, or they worked in the background.

Even though i'm not very up-to-date with web technologies(except trying out `React`&`Redux` for a few weeks), i always wanted to experiment with Static Website Generators. After investigating a little bit, i decided to make this blog using `Hugo` and `Github Pages`.

I created 2 projects called `personal-website` and `yengas.github.io`. `personal-website` included my Hugo configurations, blog posts and static assets, and the `yengas.github.io` was for hosting the output of my Hugo builds. I wanted to automate this process so i could post from anywhere(specially mobile) and my blog would automatically update.

At first, i thought about creating a `continous integration` pipeline via `Jenkins` on my `DigitalOcean` hosted VPS. But after reading more about Hugo, i found out there was a third party service called [wercker](http://www.wercker.com/) that could make this process easier, and require no setup. The only downside is that it requires full access to your Github account. I don't have any critical code on my Github account, so i decided to use wercker.

This way, i would have completely free and automated personal blog whose performance and security depends on Github Pages and my Github Account credentials.

I only had few struggles while i was trying to set this up. First of all, there isn't an up-to-date wercker script that pushes a git project to master branch of a repository. So i had to create a bash script inside of my wercker config to publish my project to gh-pages.

Also, the theme([hugo-redlounge](https://github.com/tmaiaroto/hugo-redlounge)) i decided to use didn't have a multilingual support, so i had to learn more about Hugo and customize the template to my needs. I added internationalization, a list of languages, and a feature to switch the post language if there is a translation.

If you would like to setup a similiar website, you could check out this website's [source code](https://github.com/Yengas/personal-website) on Github. You could also checkout my fork of [hugo-redlounge](https://github.com/Yengas/hugo-redlounge) theme, which has some fixes and the multilingual support i was talking about.
