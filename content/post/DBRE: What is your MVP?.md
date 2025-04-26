+++
categories = ['databases', 'dbre', 'automation']
date = '2025-04-25T20:06:00+01:00'
draft = false
tags = ['dbre','toil','starting']
title = 'DBRE: What Is Your MVP?'
+++

# Or, “a litany of errors”

The lean model for continuous improvement looks something like this:

![Lean](/images/LeanModel.png)

Where you start is a matter of some debate, as is the direction of flow. However, most people start with the idea of building a Minimum Viable Product (MVP) as an experiment; measuring the results; and then learning from the measures they recorded, which then feeds into the next build of the product.

What is an MVP for a Database Reliability Engineer (DBRE)? A good, old-fashioned manual install. A best practices manual install.

There’s lots of reading available, lots of shiny DevOps toys, lots of things to try, but if I could start again, I’d do everything we needed to do manually at first. Rushing into an automated deployment is something you will come to regret every day.

I know I have!

To my mind, automation is actually not the place to start, monitoring is. You need to think about what you want to monitor, what alerts you will need, and what constitutes success. You’ll need a basic monitoring strategy in place that covers essentials, the nice to have stuff can be added later. Have a handful of critical alerts, too, but don’t go overboard. You may find that some alarms trigger more frequently than you’d expect for reasons outside your control and fixing that can be tiresome.

I know I have!

Remember, as a DBRE, you’re looking after cattle, not pets. You care about meeting SLOs, not whether the Frammerjammer ratio is higher than the Buttersniff coefficient. You can look after the Frammerjammer ratio when all the other stuff is done, or if it becomes an issue. Don’t gather a ton of stats you’ll never look at, you will not be short of work. Also, until you have finalised your metrics taxonomy, the fewer stats you have, the fewer orphan stats you will have and the fewer changes you will have to make to all your dashboards. You will find this self-inflicted toil arduous.

I know I have!

You will also find things happening that you didn’t foresee and an initial broad “the database is borked” mechanism is much more useful day to day than the metrics you use to performance tune a single database server. I can’t emphasise this enough: there are so many moving parts in a modern application stack that it can take hours for someone in support to get to the database as a potential problem. Build something like an insert into a table, local to the database server, so if it fails and you alert on it, it greatly reduces the scope of investigation. An insert from a remote machine could fail for reasons that have nothing to do with the database. It will be a huge step forward in your work as a DBRE if you can notify people that there is an issue with the database (or not) early on in the triage process. Keep it simple: trying to build something all-singing, all-dancing when you don’t know what the problems will be is something you will come to recognise as a complete folly.

I know I have!

Don’t discount the effort of the definition of success of your experiment, either: it can be a very difficult thing to quantify, even creating a couple of SLOs can be a challenging experience, even for big organisations who put a lot of effort into it, like GitLab. You’d be surprised at how many businesses don’t actually have useful measures at all, let alone measures that you can use to define reasonable SLOs.

I know I have!

Don’t automate until you have a reasonably good idea of what you want to automate. We used a Google Form with cutting and pasting commands for daily checks, until I got irritated by the “toil” it took and then automated the checks. By that point, we probably had over 95% of our health check finalised, and automating it wasn’t shooting at a fast moving target any more. That was something we did right.

(But don’t ask me why we do daily health checks.)

It’s honestly better to keep rolling things out manually and learning that way what the issues are and then starting an automation process that fixes “toil”. Ignore the temptation to do too much upfront, or to pre-empt scenarios unless they are known, high-probability issues.

You will never automate away all your toil, but you can easily waste a lot of time automating something that isn’t proven to be toil.

I know I have!

[![IknowIhave](https://img.youtube.com/vi/T4TlVRsm8v0/0.jpg)](https://www.youtube.com/watch?v=T4TlVRsm8v0)

# Key takeaways
- Do everything manually to start off with. This will quickly tell you where the actual toil is.
- Don’t try to build a Mission Impossible command room monitoring system from the outset.
- Getting a useful definition of success that is achievable and measurable is much harder than it looks.
