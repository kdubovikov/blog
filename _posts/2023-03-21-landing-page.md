---
layout: post
title:  "Coding a SaaS landing page"
date:   2023-03-21 00:00:00
categories: programming
---

I've decided to create a side project to catch up with the latest tech trends, see what's new in frontend, and experience the thrill of setting up a small SaaS. My first step was to create a landing page, and in this post, I will share my research and results.


<!--more-->

Before starting to write the actual software, I did some planning and listed the main features of the system I plan to build using Domain-Driven Design (DDD). Then, I decided to create a small landing page to do some marketing and measure potential during the building process.

While there are many well-known solutions to set up a landing page without coding, such as Wix, I was aware that I had been out of the frontend race for quite a while, and things change fast in frontend. So, creating a landing page could be a great way to practice and catch up with the latest trends.

# Researching What's New in Frontend

## Language Choice
As someone who loves the process of learning new things, I'm always on the lookout for the latest and greatest tools in frontend development. Though it may not always be the most productive approach, I find that enjoying the development process is key when it comes to working on side projects. At first, I was captivated by [Elm](https://elmprogramming.com/), but I soon realized that it doesn't have a large following and may have some issues with integrating with the rest of the frontend development world. Additionally, I thought that using a paid template to bootstrap the app development process would be a good investment, and those templates are usually only available for major, popular frameworks.

So, I decided to go with something more industry-wide. This way, I can also refresh my skills, which may come in handy in my daily work. I have previously used all of the major Javascript UI frameworks, including React, Vue, and Angular, but it's been a while since I've used the latter two. My most recent experience with React was about two years ago when I wrote a small geodata visualization app that allowed rendering interactive heatmaps on massive datasets for one of my data science projects. Overall, I enjoyed using React for writing small and medium-sized apps, so I decided to use it again.

After working extensively with dynamically-typed languages, I've grown tired of the price you pay for code stability and maintainability in the long run. While dynamic typing is great, static type checkers have become so intelligent that they bring a lot of value with them. {% sidenote 1 "Static type checking still makes your code more verbose. In a sense, it increases your development time along with improving software stability, maintainability, and making refactoring easy. As always, it's all about tradeoffs. Dynamic typing is great as well, but there are no silver bullets." %} So, I opted for TypeScript.
{EDIT - Changes made for better readability, storytelling, grammar and SEO optimization.} 

## Framework choice

While React is only one part of the solution, I was looking for a more comprehensive framework to help me bootstrap routing, hosting, and optimization. At first, I thought about following the well-trodden path with [create-react-app](https://create-react-app.dev/), but then decided to look for a more advanced option. After a quick research, I discovered [Next.js](https://nextjs.org/), which is an opinionated framework that comes with a set of choices regarding how you structure your app and which routing you use, among other things. As with any framework, if you buy into the framework's ideology, it can give you a productivity boost.

After going through the starter course, I was impressed with Next.js's file-path-based routing. Plus, Next.js makes it easy to quickly deploy your app using [Vercel](https://vercel.com/). It's pretty open in terms of changing parts of the framework to your liking and comes with useful plugins like [authentication](https://next-auth.js.org/). 

Another great feature of Next.js is its capability to quickly create serverless APIs without leaving your frontend project by simply creating a file! These APIs get deployed automatically, making it a powerful pattern. Although I plan to use Go as my primary backend language, this feature will come in handy for quick prototyping or integration glue code.

Next.js can be easily used to build a static website generator, SPA, or a multi-page app, meaning I could use the same framework for developing my landing page and a SaaS UI later on. Overall, their approach is all-encompassing and pretty flexible, so I've decided to give it a try.

## CSS frameworks

I came to know about [tailwind CSS](https://tailwindcss.com/), which is a very interesting take on CSS frameworks compared to [Bootstrap](https://getbootstrap.com/) or [Bulma](https://bulma.io/) that I am more familiar with. Traditional CSS frameworks provide you with a set of components and well-crafted pre-defined styles that you can use to build your app. You can customize them to some extent, but they come with their own styles that can be difficult to override.

In contrast, Tailwind takes a different approach:

* No pre-defined components. If you need them, you can use [Tailwind UI](https://tailwindui.com/components) or other options that build on top of the core Tailwind library.
* You don't write separate `.css` files.
* Instead, you apply lots of pre-created inline classes that Tailwind provides.
* Follow a single source of truth approach where you can see both styling and markup from a single file.

After playing with it for a bit, I've enjoyed this approach very much. It is pretty transparent which CSS you will get compared to traditional CSS frameworks. But it still packs a decent productivity boost. The documentation is also pleasant to use. Later on, I will need to build a custom text-editor component that won't be covered by any existing component frameworks, so it will be a good investment to get familiar with this framework. Tailwind CSS is an instant win for me.

## Landing page template

As I'm not a designer, spending lots of time designing a custom landing page isn't the best use of my time. Instead, finding a template that I can modify will do. Fortunately, there are many templates available for the NextJS and Tailwind combo. For my landing page, I wanted to find something free. Here are some of the options I've saved, both free and paid. Maybe this will help you reduce your search time:

* [Tailwind Awesome](https://www.tailwindawesome.com/)
* [Tailwind UI Templates](https://tailwindui.com/templates#browse) 
* [Tailwind Landing Page Template by Cruip](https://github.com/cruip/tailwind-landing-page-template)
* [Shine - Tailwind Landing Page Template](https://preview.uideck.com/tailwind/shine/)
* [Treact - Tailwind React UI Kit](https://treact.owaiskhan.me)

## Waitlist

To add some value to the landing page, I decided to include a waitlist. While it's possible to create one yourself, I felt that my time would be better spent on the actual software. After reviewing a few solutions, I settled on [GetWaitlist](https://getwaitlist.com). Integrating it into the landing page was simple, and the setup was fast.

# The Final Result

You can see the final result here: [https://hermesprompt.com/](https://hermesprompt.com/). While it may not be the best landing page ever, that wasn't the goal. Subjectively, it's better than many landing pages for funded startups that I've seen. In my next post, I'll focus on generating ideas and building the actual product. If you are interested, join the waitlist!
