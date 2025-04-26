+++
categories = ['databases', 'dbre']
date = '2025-04-26T09:51:11+01:00'
draft = false
title = 'The Forgotten Aspect of DBRE'
tags = ['dbre', 'upgrades', 'old posts']
+++

---

# The Forgotten Aspect of DBRE

![The horror... the horror...](/images/TheHorror.jpg)

I am a great fan of the DBRE book, and an avid follower of the authors' podcasts, videos and blog posts.

But there is one glaring omission that I can't find discussed properly anywhere: upgrades.

You need to patch to fix bugs or address vulnerabilities. You need to upgrade for support and new features that developers want.

I understand that site-specific needs make it hard to proclaim detailed edicts. But the DBRE book isn't about detailed edicts. It's about principles and a way of working.

# Strategy

There should be a clear strategy around patching and upgrading. I want to call out some things to consider:
- all too few products have a clear version strategy. If they do, it should drive your approach
- for products that have an unclear strategy, you will have to track their releases by hand
- deploy security patches immediately (if appropriate)
- deploy bug fixes at least quarterly (if appropriate)
- if you have less than two years' support left, start testing a newer version now
- if you have less than one year's support left, start rolling out a new version now
- start planning and engineering the rollout now. A developer need to upgrade can happen at any time
- plan for an annual major upgrade. It's better to have everything in place and not use it than have to cobble something together every time

A great example of a clear version strategy is [PostgreSQL](https://www.postgresql.org/support/versioning/). I am disappointed that more databases do not have such a clear and predictable process:
> Minor releases only fix bugs and vulnerabilities to reduce risk. For minor releases, the community considers not upgrading to be riskier than upgrading.

It's worth adopting their versioning strategy in full if you are a database vendor. Hint, hint!

# Testing

This is the most painful part of upgrading. Nobody wants to replace something that works with an unknown quantity. The only way to reassure people is through rigorous testing.

# Minor Releases in a Clear Version Strategy

Apply minor releases to all pre-production environments at once. If no issues surface within a given period, roll out the minor release to production.

# No Clear Version Strategy or Major Version Upgrade

This approach applies to a minor release with no clear version strategy.

It also applies to a major upgrade irrespective of strategy.

If your developers use Docker on their workstation, start there. Replace the current version with the new version and run for a given period.

If no problems arise, roll out to build, integration, NFT and QA environments for a given period.

If no problems arise, roll out to production.

How long is "a given period"? That's up to you! It may be as little as one test run, if you have great test coverage. I suspect most teams would want at least a sprint in each environment. Your mileage may vary, etc.

# Engineering

You should build database upgrades (and downgrades!) into your CI/CD pipelines from the outset. We all know that retrofitting things is more difficult.

Use the concept of a "feature switch" to determine which version to deploy.

Rolling back a version upgrade can be a challenge. It is especially difficult (impossible?) when using certain new features. If the upgrade breaks backward compatibility, your tests must be comprehensive.

# Conclusion

Upgrades are painful, but in the world of DRE, they shouldn't be. They're painful because they aren't regular and they're not automated. Your integration and NFT should include running different database versions.

Taking a PostgreSQL example, assume you are running 9.6. You could have a parallel integration for 10, 11 and 12. If you upgrade every year, you will have been testing 12 for 3 years before you adopt it. Even if you adopt it sooner, you will have tested it better than if you waited till adoption to start testing.

Other products have less clear strategies. But there is no reason you can't test every "future" version as a normal part of your integration. Disqualifying bad releases long before adoption saves a lot of pain.

I don't have all (or even most) of the answers. Comments and suggestions are welcome!

---

# Key takeaway:

Upgrades are painful, but in the world of DRE, they shouldn't be. They're painful because they aren't regular and they're not automated. Your integration and NFT should include running different database versions.
