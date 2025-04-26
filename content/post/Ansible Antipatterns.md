+++
categories = ['databases', 'dbre', 'automation']
date = '2025-04-26T10:32:52+01:00'
draft = false
title = 'Ansible Antipatterns'
tags = ['dbre', 'ansible', 'old posts', 'automation']
+++

# Ansible Antipatterns

Lots of people tell you how to do things properly. Here are some things I've inherited in our Ansible that break things in horrible ways.

![Workshop](/images/carlos-irineu-da-costa-eMc0lpn1P60-unsplash.jpg)

# Coupling

Coupling is (roughly) how interconnected different components of a system are. One of the most common patterns in modern software engineering is loose coupling.

The idea of separating things across an API so that you can change things behind the API without having to change other components is very powerful.

Conversely, the antipattern of combining everything tightly is easier for the initial development because you don't have to design and implement the API, however, the maintenance becomes a nightmare.

Because our software is multi-tier but pretty much everything needs to know about the Informix databases, someone took the decision to embed the Informix installation into our infrastructure build. This has led to all sorts of interesting issues in our attempts to improve it:
- changing the Ansible becomes more difficult, because there is more of it
- surprising "dependency" issues come up, like accidentally using variable names used in some other Ansible, breaking a component that you didn't even know about
- changing the Ansible breaks a tightly-coupled feature, so you have to change that too
- breaking the Ansible out of the infrastructure build becomes more difficult every time you look at it
- adding separate Ansible to do something ancillary to a database becomes more difficult, because you have to figure out where and how to fit it into the whole shebang
- testing and deploying takes a long time

Even if you have something like a standalone MySQL installation embedded into the CMS installation, it causes a lot of the same problems.

# Versioning / Packaging

This is a contentious one. I don't mean that it's a bad thing to version database builds, but the decision to tie a product release to a specific version of the database leads to some horrible issues.

For example, we have (like most people) things that we fix in our builds to manage space better, avoid split brains, etc., etc. Because our releases contain a specific database build version, it means that something critical that we've fixed in a newer build can take months to get deployed.

In the meantime, we have to fix that issue manually and even worse, if a site is a 2 or 3 releases behind for some or other reason, then we have to remember to fix it over and over again.

If there is a reason to keep a specific deployment version tied to a release, then make sure you have bunch of things that can be deployed irrespective of the version. How you square that circle is a non-trivial issue though.

# Structuring / Consistency

Unlike a conventional programming language, Ansible does not have the idea of functions. Well, it kind of does, but using them is considered an antipattern. This leads to a massive amount of code duplication and inconsistency.

Rather than using a standard bit of code with a variable to do logging, for instance, the standard practice is to do this individually per role or playbook. It makes things like log trawling really difficult, because of the lack of consistency. It also makes maintenance more difficult because a change to something used all over the place has to be changed all over the place.

I disagree with this, I think having a "library" of standard "functions" will greatly improve how your code looks and behaves. It will also make your playbooks shorter and more concise. If you treat it like an API, then it should also hugely improve maintainability.

However, there's a lot of debate over how to structure Ansible in general. What should an individual playbook contain? Where are the boundaries? The advice I was given was that "all the Ansible necessary to do a thing" should be in one playbook.

But your idea of what "a thing" is and mine will almost certainly differ. This makes maintenance difficult, because you may have have to jump all over the place looking for code. Also, because all the Ansible necessary to do a thing may include a lot of things that could be in a common library, the playbooks will be bigger than they need to be.

Trying to make this decision organically is a big antipattern, spend some time up front to work out a policy and enforce it rigorously.

# Using Ansible without understanding the (whole) problem

In retrospect, I wish we hadn't used Ansible for everything from the start. We could quite easily have deployed Informix (and all the other databases) by hand and I'm not sure they would have been any less consistent than they already are.

Because of the versioning issue mentioned above, we have different installations of Informix across sites anyway. We could very easily have used the Informix installer to create a "standard" install and spent our engineering time building scripts and Ansible that manages the the environment better. This would have dramatically simplified the Ansible and made much better use of our time.

Also, while I am a huge fan of the fact that I can deploy an Informix cluster hands-free in half an hour or whatever, running Ansible in production environments with things like our authentication tool is incredibly slow and leads to serious outages for upgrades / deployments. I'm not blaming Ansible or the authentication tool, but the interaction between the two is an issue. Or maybe it's something to do with VMWare, I don't know. (Your mileage may vary.)

I don't know what the answer to this is for us, but I suspect something like AMIs / Docker with Kubernetes[^1] might be.

However, if we'd done stuff manually from the outset, we might have gone (more or less) straight there, before we'd invested so heavily in Ansible.

It sounds counterintuitive for an experienced DBA to say this, but understanding where your toil actually is before you start automating it away is vital. By going with automation before we understood what some of the consequences would be, we've actually created some toil for ourselves!

[^1]: Update based on recent experience: Docker? Almost certainly. Kubernetes? Not so much, maybe even not at all!
