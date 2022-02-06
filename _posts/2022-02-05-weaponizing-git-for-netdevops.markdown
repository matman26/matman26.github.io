---
layout: default
title:  "Git: Decentralized Version Control for Network Automation"
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

![Figure: Email Patch Workflow](/assets/images/vc-email-workflow.svg)

There must be a better way, right!?

## CVS
CVS or Concurrent Version System was one of the first influential Version
Control Systems to be implemented. CVS' initial idea was to simplify cooperation
between small teams by allowing users to synchronize their changes with a remote
repository; CVS implements a client-server architecture where a centralized
server represents the latest state of the code. This simplified the process
of sending diffs in tarballs: developers could now focus on staying in sync with
their CVS server instead on relying on e-mails and manual patching!

While sending e-mail patches was a viable way to keep a history of changes done 
to a project, reviewing code from an e-mail client was often not ideal. CVS 
streamlined the code review process by centralizing all history on the remote repository
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


## Distributed vs. Centralized Version Control
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

## Git
### Git Basics
While Git was not even close to being the first DVCS to be, it was certainly the
most influential. Git was the result of ongoing development issues during work
on the Linux Kernel, one the largest Open Source projects to be. Linus Torvalds himself
states that he made it his philosophy to go against everything CVS chose to do from
a design standpoint when building Git: where CVS was centralized, Git would be distributed,
where CVS would be non-atomic, git would provide atomic operations. While this view can
be seen as radical, as CVS had several correct ideas for its time, it can be said that this 
decision worked to Git's benefit as it made for a much more robust solution 
when compared to CVS.

Git sees and compiles all changes made to the code as _commits_; the concept of commit has existed
ever since early VCSs, but they have a different meaning when it comes to distributed
version control. In CVS, a commit required network connectivity to the repository. In
fact, most tasks like branching, commiting and merging code required connectivity to
the central repository in CVS. Git decides instead to define commits as a set of changes
applied to the code (i.e. a **diff**) plus some metadata; these changes are linked not
to a remote repository but to a local one. Among the metadata Git links to each commit,
we can mention:
+ A commit hash which can be seen as a pointer to the commit on the tree
+ The commit author (the one who made and commited the changes)
+ A timestamp (date and time when the commit was made)
+ A message field that the commit author can use to explain changes that were made

<++> Results of git log with info here <++>

From a high-level point of view, there are three main components to Git's distributed
architecture:
+ The local workspace: actual set of files currently being manipulated on your filesystem;
+ The local repository: your local version of the git repository, with the whole history from the project synced from the remote repository;
+ The remote repository: usually a server all members of the team have access to, used to publish work and do code reviews. Services such as GitHub and GitLab usually play this role.

Whenever you modify files in your local filesystem, these changes exist temporarily
in your _workspace_: they are not yet committed to Git. Once you choose to save your
changes as a commit, you first explicity tell git which files you want to add to
your next commit using `git add <filename>` and once this is done you commit them
to your local repository using `git commit -m <message>`. 

Since all operations you do on git need to first pass through your local repository,
you can be sure that the performance of commits, branches and merges is only limited
by your filesystem; you don't need to consult any external server over the network
to do any of these, so they happen really fast.

### Branching and Merging
Branches on Git are designed to encapsulate modular changes to the codebase.
Let's suppose you have an automation script that creates VLANs on a Nexus Switch.
Your script
currently adds a Vlan ID and a description, but you'd also like it to add that 
newly-created VLAN to your trunk ports automatically; this can be seen as a 
new self-contained feature for your script. In this case, the usual workflow 
would be to sync up to the latest version of your script and then create a 
new (local) branch with a descriptive name such as `assign-vlan-to-trunk`. 
Different teams can use different conventions to name feature branches, but 
you usually want to be short while also giving any future reviewer an 
idea of what you are attempting to do with your branch.

<++> Feature Branch for assign-vlan-to-trunk <++>

```
# Set your workspace to track the default branch (usually is master)
git checkout master

# Fetch all git metadata from remote repository
git fetch

# Update local repository if any new changes are present on remote
git pull

# Creates new branch on local repository (based off of master)
git branch assign-vlan-to-trunk      

# Put local assign-vlan-to-trunk into the current workspace
git checkout assign-vlan-to-trunk    

<start working on your code here>
```

You've modified your script as needed and it now supports the new feature!
If you use `git status` you'll see that git noticed your script was
modified. You can `git add myscript.py` to move it to your staging area;
whenever you issue a commit, all changes to files in your staging area are packed
as a commit and given a unique hash along with all the other metadata.
The commit is then added to your local repository. If you feel like your
work is ready to be shared with your team-mates, you can `git push` this
code back to your remote repository.

```
# Pushes your current assign-vlan-to-trunk to remote (usually named origin).
# On the remote, create a branch called vlan-to-trunk
git push --set-upstream origin vlan-to-trunk
```

Since your local repository is now ahead of the remote one by a few commits,
and has a branch that does not exist yet in remotely,
whenever you issue a `git push` your changes will be sent over the network to
the server. In this case, we are updating the remote server called **origin** with a new branch
that we are calling **vlan-to-trunk**; notice we are creating a new branch on the
remote server with a different name from the one we have locally. Internally
Git keeps track of which local branches refer to which remote branches, so
these name differences don't pose an issue. If ou don't specify
a new branch name when pushing you'll just replicate your local branch name on
the remote if it does not exist yet.

Once your changes exist as a new branch on the remote, you can tell your team
members the feature is implemented. This is when the code review process can begin;

## Git Workflows
### Gitflow
### Continuous Integration/Continuous Delivery
