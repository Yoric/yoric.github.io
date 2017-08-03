+++
title = "Project Lighthouse: A post-mortem"
date = "2017-02-20T10:44:33+01:00"
draft = false
categories = ["Mozilla", "IoT", "post-mortem"]
tags = ["Mozilla", "Connected Devices", "IoT", "WoT", "vision-impaired",
  "computer vision", "OpenCV", "Raspberry Pi", "Raspbian", "Docker",
  "Python", "C++", "Swift", "Obj-C", "iOS", "iPhone", "wearable",
  "prototype"]
description = ""

+++

A few weeks ago, Mozilla pulled the plug on its Connected Devices Experiment:
a bunch of internal non-profit hardware-related startups. One of our main
objectives was to determine if we could come up with designs that could help
turn the tide against the spyware-riddled and gruyère-level security devices that are
currently being offered (or pushed) to unwary users.

One of the startups was [**Project Lighthouse**](https://wiki.mozilla.org/Connected_Devices/Projects/Project_Lighthouse). We tried to provide an
affordable, simple and privacy-friendly tool for people suffering from
vision impairment and who needed help in their daily life. While Mozilla
closed the experiment before we could release anything useful, we still came
up with a few nifty ideas. Here is a post-mortem, in the hope that, maybe,
someone will be able to succeed where we couldn't.

# The problems at hand

If you suffer from sever enough vision impairment, there are many things that
you can do on your own, but also many things for which you need help. Things
such as interacting with printed text, with identical-to-the-touch bottles or
cereal boxes, with the color of your clothes, etc.

The World Health Organization estimates that there are ~285 millions of
vision-impaired people in the world, including ~30 millions who are entirely
blind. Roughly 85% of them live below poverty level, which means that they
cannot afford a helper (unless it's a family member), nor an expensive gadget.

So, how can we help them?

# Real-life text recognition

At first, we considered helping vision-impaired people interact with printed text.
The general idea was to develop a camera that could read out the text from an
image, and then iterate on accessibility and form factor.

I have written in details about the difficulties of the operation in
[another blog entry](../extracting-text-from-images).

Among the questions at hand:

- is it feasible?
- would such a device need an Internet connection?
- how much CPU/memory would the device need?
- what accuracy could we expect?
- would it be limited to English?

## Results

We concluded that we could finish implementing a reasonable text extraction
with a few more weeks of work. This extraction would work just as well if the
object was held upside-down or in diagonal, and should, in theory, work with
all the common fonts used for indo-european languages. We suspected that the
algorithms would need additional work for semitic or asian languages.

We also realized that the underlying algorithms relied upon a number of magic
constants (e.g. contrast) and that we would need additional work to tune these
constants to actual use cases. We could not rely upon stock pictures, for
instance, as we could expect that vision-impaired users would not be sensitive
the lighting conditions, or shadows or the framing in the same manner as
professional or even casual photographers. It was not clear how we could capture
a sample sufficiently large to extract relevant constants.

We benchmarked that text extraction needed lots of CPU, so would probably rely
either upon a cloud service (bad for privacy, expensive for many users) or upon
a beefy cellphone (expensive for many users).

Further, we determined that extraction would be unfeasible when the orientation
of a single line of text changed too much, e.g. on round pill bottles, or on
bags of potato chips.

Finally, it was clear that extracting texts and recognizing logos were two very
different issues. No amount of text extraction would be able to parse the
calligraphy of the Coca-Cola logo, or to understand that a "G" with a specific
color combination could mean "Google", or that "moz://a" could mean "Mozilla".

## Conclusions

We decided that text extraction should not be the main feature of the device,
but rather something that we would add at a later stage, probably after logo
recognition.

# Object tagging

A second avenue was helping vision-impaired users interact with objects that
are difficult to differentiate by touch.


We envisioned three possible ways to proceed in this direction:

1. let the user take pictures of the objects, recognize what the object was;
2. let the user take pictures of the objects, record a description (
  "noodles, they need to cook 10 minutes",
  "hat, goes well with my bowtie", ...), use computer vision algorithms
  to later recognize objects from the internal database;
3. let the user take pictures of the objects, record a description, use
  deep learning to later recognize objects from the internal database.

We decided to avoid 1. for a minimal viable product, as it would require either a large
database and a cloud service, or reliance upon third-party cloud services,
which in turn meant:

- dependency on an always-present internet connection;
- many potential privacy issues.

We also decided to avoid 3. for a minimal viable product, as we did not have
the in-team skills to proceed with deep learning. We meant to return to 3. in
a relatively close future, though.

So, we went ahead with option 2. Among the questions at hand:

- is it feasible?
- would such a device need an Internet connection?
- how much CPU/memory would the device need?
- what accuracy could we expect?

## Results

After a few weeks of coding, we managed to produce a pretty nice prototype,
using a Raspberry Pi, OpenCV and Python. In this limited span of time, we did
not seriously attempt to resolve issues such as shaky cameras, but otherwise,
the prototype worked nicely. Recording or finding an object in the database was
nearly instantaneous (for a database of a dozen recorded objects only), without
an internet connection, and accuracy was good (we did not attempt to quantify it).

We needed to come up with a method to let the camera differentiate the object
from its background. In the absence of sophisticated stereoscopic cameras, we
used two tricks:

- ask the user to show the object, then hide it; or
- ask the user to shake the object in from of the camera.

We tested some of the prototypes with real users, got good feedback
and figured that we could iterate from there. Our prototypes cost ~$50 to build,
a far cry from the $1,500+ we found for other devices for vision-impaired users.

Also, other tests with a few fully blind users confirmed that aiming with a
camera was not a problem. There were difficulties with judging distance or
light, of course, but we expected that these could be mitigated by judicious
algorithms.

So, we decided to proceed from this prototype.

# Form factor

Once we were reasonably convinced that we could come up with a technology to
make the device work, we started working on the form factor. We came up with
several ideas:

- a camera in a necklace;
- a camera attached to the user's glasses;
- a camera that could attach to many places, including a supermarket trolley.

Unfortunately, the project was canceled before we could make much headway in
the form factor.

An important lesson, though, is that going from design to industrial prototype
takes lots of time. I do not have hard numbers, as we didn't reach that stage,
but my personal estimate puts this around **6 months**. Which means that we
needed a way to test our code and our form factor prototypes with real users.

We did not get the opportunity to test the form factor, although [I discussed in
another post](how-to-survive.html) some of the techniques that we could have
used for this purpose.

# App

It would have been unrealistic to wait for the industrial prototypes before
proceeding, so we came up with an idea to test our code in parallel with the
external and industrial design.

The idea was to port our algorithms to a device that a sufficiently large number
of our user base already owned. While developing for Android would have been
nice, our statistics (US only) showed almost all vision-impaired users who owned
a smartphone were using iPhones, so we did not have much of a choice. We decided
to develop an application for iPhone, release it first on Apple's TestFlight
(for alpha-testers), then the App Store.

Additionally, we expected that some users who already owned an iPhone could be
interested in the application, even without an optimal form factor. For other users, we still intended to provide an
inexpensive full-featured device.

## Results

We managed to port our code relatively easily from Linux to iOS but the project
was canceled while we were still 1-2 weeks away from a first pre-alpha release,
an estimated 4-6 weeks before we could go beta.

Performance was excellent, which is no surprise as iPhones are largely more
powerful than Raspberry Pis.

While population numbers make it clear that iOS was the right platform for early
testing and would be the only reasonable target for a smartphone-only application
for vision-impaired people *in the US*, we had a terrible experience developing
for iPhone.

Since we needed to interact with a C++ library for computer vision, we needed
to code in three languages: C++ (for the computer vision), Swift (for the UX)
and Obj-C++ (the unnatural grandchild of C), with two different threading models
and three different memory management paradigms (C, C++, Swift/Obj-C). Getting
to a stage at which we could perform callbacks from C++ to Swift took some
serious digging and understanding Obj-C++'s very *interesting* memory model.

The signature mechanism seems to be designed largely to get developers to pay
(either $99 per developer or $199 per team), which is quite adverse to both
open-source contributions and to continuous integration. It also makes it
hard/impossible to distribute binaries for testing, even to your non-developer
teammates.

Also, the command-line tools offer now reliable way to launch unit/integration
tests, plus they have breaking syntax changes – these two points contribute to
making continuous integration needlessly difficult.

# Not tested: Crowdsourcing help

One of the features we were considering was a way to crowdsource requests for
assistance. Consider a vision-impaired user with a question that cannot be
easily answered by a computer: "My arm hurts, is there anything strange on it?",
"Does this dress match this hat?", etc. How can we put the user in touch with
someone willing to help?

Broadly speaking, our idea was the following:

- extend the camera device and application with the ability to record a question;
- develop a web service to gather the questions, their answers, moderation, meta-moderation, ...;
- develop a web application front-end to the web service;
- wrap this web application as smartphone apps and browser add-ons for all major
browsers, designed so that a user could spend ~5-20 seconds helping whenever they
felt like it.

This left many issues unanswered, such as how to avoid spamming users with
inappropriate content, how to handle privacy ("I have photographed my credit
card by accident"), etc.

We only had time for the first paper explorations of the topic, but we
believed that such an open-source crowdsourcing platform for requesting
5-20 seconds help could have become very useful for many projects.

