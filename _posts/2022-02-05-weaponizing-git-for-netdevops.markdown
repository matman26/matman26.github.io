---
layout: default
title:  "Git: Weaponizing Version Control for NetDevOps"
date:   2022-02-05 12:00:00 -0300
permalink: /posts/weaponizing-git-for-netdevops
image: /assets/images/distributed-version-control.svg
---

# Version Control
It is often very easy to underestimate the value of good version control 
for software development. Git, in particular, is one skill that is often
overlooked on the Cisco DevNet Associate (DEVASC) Exam blueprint. It's not too
hard to see why: most of the training courses aimed at Network Engineers
often miss the point of why Git (and version control in general) are
crucial parts of successful software development (this includes, evidently,
automation).

![Figure: A common meme about Git](/assets/images/git-meme.png)

When teaching Git, it is very easy to try to give students a cheat-sheet
with a few commands teaching how to do a `git commit`, a `git push` and
so on. This does explain **how** git works but aspiring Network Automation
Engineers need to focus just as much on **why** these workings are important
pieces of automation in their own right; this reasoning is what we'll be 
taking a look on this article. We'll see that Git is a very intuitive
piece of software once you understand a few concepts. Once we get there,
we can focus on what we want to do with our versioning, rather than
which CLI commands and options we should be using; this is when we can use
the tool to its fullest potential.

Going back to the basics, Version Control is the process of versioning the
code that powers our application. Version control has been done in one way or another
ever since the beginning of programming itself, and has become more and
more important with time as software development teams grow larger and
projects become more complex. Whenever you, as a member of a team,
apply some changes to the source code of your project, you need a 
standardized way to keep track of older versions so you can quickly
rollback and also a way to communicate changes to your teammates
so that everyone is aware of what the most recent version of your
code is.

The most important features a source control system needs are:
+ Maintain a history of changes for all relevant files on the project so as to allow rollbacks;
+ Allow multiple developers to work on a single code-base while keeping changes consistent across all team members;
+ Resolve simple conflicts that can arise when developers work on the same set of files simultaneously;
+ Allow team members to trace back when and by whom a specific change was introduced to the code;
+ Maintain different branches (versions) of your code for production and testing purposes.

![Figure: Pillars of Version Control](/assets/images/svc-pillars.svg)

Historically, version control has been achieved in several different ways, with varying
degrees of success. Linus Torvalds, the creator of Linux (and also the creator
of Git, we'll get to that) has said during an interview that the development of
the Linux Kernel was handled by Patch files being sent in a tarball format: every
time someone on the team made some change to the code, she would generate a diff
between the original code and the new version, compress that as a **tar.gz**
archive and send it as an e-mail attachment to other developers. This, of course,
was very labor intensive and prone to human errors, as patches were manually 
generated and manually applied.

![Figure: Email Patch Workflow](/assets/images/email-patch-workflow.svg)

There must be a better way, right?

# CVS
CVS or Concurrent Version System was one of the first influential Version
Control Systems to be implemented. CVS' initial idea was to simplify cooperation
between small teams by allowing users to syncrhonize their changes with a remote
repository; CVS implements a client-server architecture where a centralized
server represents the latest state of the Code. This simplified the process
of sending diffs in tarballs: developers could now focus on staying in sync with
their CVS server instead on relying on e-mails and manual patching!

While sending e-mail patches was a viable way to keep a history of changes done 
to a project, reviewing code from an e-mail client was often not ideal. CVS 
streamlined this process by centralizing all history on the remote repository
and allowing developers to access its history log. CVS could be used as a CLI
application to do operations such as `commit` to update
the remote repository with new code and `update` to sync-up with the latest
version available on the server. The CVS client would automate much of the
patching process, and was usually smart enough to resolve simple 
conflicts automatically if it found the same file was update by two 
different developers.

![Figure: CVS Workflow](/assets/images/cvs-workflow.svg)

Of course, the fact that CVS relied on a central server meant that there 
was only one actual valid state for the codebase to be in; developers did
not have a local copy of the metadata on the CVS repository, which meant that
if the CVS server was somehow compromised all history data would be lost. Due
to the client-server model, CVS also required developers to have network
connectivity to the server, making it impossible to generate commits offline
and sync them up at a later moment. CVS also didn't track code patching
atomically; this meant that if a patch failed due to some outside 
event, the repository could be left in a corrupted and unusable state.


# Distributed vs. Centralized Version Control
CVS was definitely a step in the right direction conceptually when 
compared to manually generating and sending patches, but it also had
its fair share of flaws.

<+++++> More stuff here <+++++>

![Figure: Centralized Version Control](/assets/images/centralized-version-control.svg)

The main difference between a centralized and a distributed version
control system is that there is no single source-of-truth containing
all metadata for the project on distributed systems. Each developer is assigned their own local
repository that behaves exactly how a remote one would behave:
you can commit changes to your local repository and sync up from it. The
distributed model has the clear advantage that you do not rely on data from
the remote server and can thus commit and sync-up offline. You only need
a network connection to your remote server when you sync up your local
repository to the remote one via `pull` to grab new changes and `push`
to publish your local changes. This also means that branching can be
done completely offline and becomes potentially transparent to 
other team members. You can commit incomplete drafts to your local repository
so as to generate a rollback point, and only publish those changes
later when they are fully finished.

![Figure: Distributed Version Control](/assets/images/distributed-version-control.svg)

Since every member of the team has their own copy of the codebase's repository,
there is no single-point-of-failure. If the remote
server is somehow compromised, it can still be recovered via someone else's
local repository; all repositories, local and remote, are equals in this regard.

# Git Basics
While Git was not even close to being the first DVCS to be, it was certainly the
most influential. Git was the result of ongoing development issues during work
on the Linux Kernel, one the largest Open Source projects to be. Linus Torvalds himself
states that he made it his philosophy to go against everything CVS chose to do from
a design standpoint when building Git: where CVS was centralized, Git would be distributed,
where CVS would be non-atomic, git would provide atomic operations. While this view can
be seen as radical, as CVS had several correct ideas for its time, it can be said that this 
decision worked to Git's benefit as it made for a much more robust solution 
when compared to CVS.

# Git Workflows
## Gitflow
## Continuous Integration/Continuous Delivery
