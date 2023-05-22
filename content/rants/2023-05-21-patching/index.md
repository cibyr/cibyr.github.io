+++
title = "Patching Is An Anti-pattern"
date = 2023-05-21
+++

# Patching Is An Anti-pattern

## Security updates are necessary and good

If you're professionally responsible for one or more computers, part of that responsibility likely includes installing security updates. You probably have some sort of mandate from your CISO or security department (if you have either of those) to install security updates within a specified timeframe, and maybe even some reporting on how well you're actually keeping up with that. This is effectively just a requirement of being on the internet these days. Sure, people get worked up about 0-days and APTs, but [the most basic thing that's going to ruin your day is some workaday ransomware crew exploiting years-old vulnerabilities](https://www.cpomagazine.com/cyber-security/new-study-finds-that-ransomware-attacks-are-heavily-relying-on-old-vulnerabilities-unpatched-issues-dating-back-to-2010-still-exploited/). Running old, vulnerable software is just asking for trouble; the internet equivalent of wearing a "kick me" sign on the school playground.

Hence device management tools that nag users, remotely install updates, report on compliance, and even block network access as required to get updates installed. This all makes sense for traditional IT - laptops, desktops, smartphones and such; probably even for physical servers if you're unlucky enough to still have any of those. But, I'm here to tell you, it does _not_ make any sense in the cloud. In fact, it's a sign that something is horribly wrong with how you build, test, and deploy software.


## The cloud is magical

You - yes _you_, dear reader - have the ability to conjure computers into existence in mere seconds. That's what `aws ec2 run-instances` does, and all the other cloud providers have something equivalent. Even if you're running on-premises servers, you're probably not running them on bare metal - virtualization platforms like VMware make life so much easier you've either got to be a masochist or doing something special like HPC to be bothered. So servers are something that come and go at your whims. You might even have automation that launches new servers when you need to scale up under load, and shuts them down again when it gets quiet.

But that's not even the magical part. The _really_ cool thing is that when you create a virtual machine, you specify the contents of its boot volume. You get to choose exactly what software it runs, right from the get-go! This is amazingly powerful. If you (or your build system) creates a disk image with your software and all its dependencies already installed, a lot of things get better:

* Deployments are now just creating machines with your new image, and shutting down the old ones
* Rollbacks are the same, but with the old version instead
* Scaling up is quick and painless - just run more machines!

All of this is just as true of containers as well - a container image is really just a filesystem image composed of copy-on-write layers.

### But I have state on disk!

Yeah yeah, so do I. Don't put it on your boot volume. Obviously it's heaps more convenient if your state is in some database (or object storage service) you talk to over the network, but even if you have some good reason to put your state in files on a local filesystem, you can still follow this pattern.

If you're using network-attached block storage like EBS, then it's easy: keep your data on a second volume. During deployments you take the data volume from your old machine, attach it to your new machine, and you're good to go!

Even if you're using local storage, you can [replace your boot volume](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/replace-root.html) without losing your local state.


## Or you could do it the hard way

A lot of systems I've seen do something else instead. They launch machines with a "base image" that contains little more than an OS. Then, upon first boot, the OS installs security updates (possibly rebooting along the way if this includes a new kernel). Once it's done with that, it downloads the application that it's supposed to be running (and whatever dependencies are required) from somewhere. Only then (maybe minutes later!) can it start running the code that is the sole reason it was brought into existence to run. Usually there's separate processes to deploy (and rollback) updates to the application, and often a third process to trigger OS security updates.

This _sucks_. It makes scaling up slower (which in turn means the system needs to operate with a larger buffer, increasing costs), and introduces the possibility that freshly-launched machines might behave differently to existing ones that have just received a deployment. But it gets worse...

### Do you like testing in production?

Did you catch that part about the OS installing updates on boot? I skipped an important detail there: what version of the OS packages does it update _to_? Well, they're security updates, so you probably need the latest—

Oops! Now when you launch a new machine, you're potentially creating a combination of versions (OS packages and your application code) that has never existed anywhere else before. That's probably fine if that happens in your development or testing environment, but if you're scaling up in production, this is scary! One of those updates might have introduced a horrible performance regression for your application. It's even possible for an update to break an assumption that your deployment scripts were relying on - preventing your application from starting at all!

This isn't a hypothetical. I've seen systems like this run from years with no apparent problems, and then suddenly the team responsible finds themselves unable to deploy, scale up, or even rollback, worldwide! The one saving grace is so long as you don't touch _anything_, servers that are currently running will probably stay running. What a nightmare!

{{ image(sources=["testing-in-prod.jpg"], fallback_path="testing-in-prod.jpg", fallback_alt="I don't always test my code, but when I do, I do it in production") }}

You can avoid this disaster by pinning the version of updates that get installed, and rolling out changes to that version pin with the same care that you would deploy updates to your own code. This seems like a half-measure to me: now you're going to all the effort of versioning, testing, and deploying updates to your OS but without the benefits of having those updates baked into the image you run machines with.


## Security updates are just changes you want to deploy

Just like updating a version of a library you depend on, deploying OS updates should be a straightforward, controlled, process:

1. A human (or ideally, some automation) notices a new version has been published and updates the source-controlled reference that tells your build system where to get it.

2. Your build system assembles it with the rest of your software into an artifact that can be deployed to production (e.g., a machine or container image).

    - Probably also runs some unit tests, but that's not particularly relevant to this story.

3. You (or some automation) deploy that new image to your test environment(s) and check that it works.

4. Then gradually deploy it to your production environments, monitoring for issues as it goes.

Just like that, you're no longer running old, vulnerable software yet you haven't patched a single machine! Even better, if something _does_ go wrong, you have an obvious, straightforward way to roll it back.

So say it with me now:

# Patching👏Is👏An👏Anti👏Pattern👏