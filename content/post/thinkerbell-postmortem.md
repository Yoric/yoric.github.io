---
title: "Thinkerbell Postmortem/Brain dump"
date: 2018-03-08T08:50:05+01:00
lastmod: 2018-03-08T08:50:05+01:00
draft: true
keywords: ["programming language", "DSL", "types", "process algebra", "IoT"]
description: "A programming language for the SmartHome. Also, a type system and a process algebra."
tags: ["programming language", "DSL", "types", "process algebra", "IoT"]
categories: ["research", "mozilla", "IoT"]
author: "David Teller"
---

Two years ago, I was working on a research project called "Project Link" as part of
the Connected Devices branch of Mozilla. While this branch has since been stopped,
some part of Project Link lives on as [Project Things](https://blog.mozilla.org/blog/2018/02/06/announcing-project-things-open-framework-connecting-devices-web/).

One of the parts of Project Link that hasn't made it to Project Things (so far)
was Thinkerbell: a Domain-Specific Language designed to let users program their SmartHome
without coding. While only parts of Thinkerbell were ever implemented, they were sufficient
to write programs such as:

> Whenever I press any button labelled "light" in the living room, toggle all the lights in the living room.

or

> If the entry door is locked and the motion detector notices motion, send an alarm to my SmartPhone

Thinkerbell also had:

- semantics that ensured that scripts could continue/resume running unmonitored even when hardware
    was replaced/upgraded/moved around the house, including both the server and the sensors;
- a visual syntax, rather than a text syntax;
- a novel type system designed to avoid physical accidents;
- a semantics based on process algebras.

Ideally, I'd like to take the time to write a research paper on Thinkerbell, but
realistically, there is very little chance that I'll find that time.
So, rather than letting these ideas die in some corner of my brain, here is a post-mortem
for Thinkerbell, in the hope that someone, somewhere, will pick some of the stuff and gives
it a second life.

Note that some of the ideas exposed here were never actually implemented.

<!--more-->

# Scripts

Let us consider the kind of scripts we wanted to write:

> Whenever I press any button labelled "light" in the living room, toggle all the lights in the living room.

or

> If the entry door is locked and the motion detector notices motion, send an alarm to my SmartPhone.

or

> When the motion detector of the living room hasn't noticed motion for 30 seconds, turn off all the lights in the living room.

or

> If the entry door is locked and the motion detector notices motion in the living room, if I have a camera in the living room, send an picture to my SmartPhone.

All these scripts were meant to be executed on a server hosted inside the home (for both privacy
and reliability reasons). The server itself would also run (mostly trusted) device drivers that
would let scripts access connected devices such as lightbulbs, water faucets, fridges, ... These
same device drivers also provided access to remote applications, through a mechanism of authorizations
and tunneling that has been ported to Project Things (and wasn't part of Thinkerbell itself).

## Text syntax

We never attempted to design a human-readable text syntax, as Thinkerbell was designed for
visual programming. Indeed, we implemented a visual interface for programming without
writing, although what we had was for demonstration purposes and was much more limited
thank Thinkerbell itself.

So, anything below is actually the JSON syntax designed to serve as back-end for the UX.

> Whenever I press any button labelled "light" in the living room, toggle all the lights in the living room.

```js
{
    // Rules are triggered when a set of conditions becomes true.
    // These conditions are provided by device drivers, aka `source`s.
    "conditions": [
        {
            // In Thinkerbell, we almost never specify a single device.
            // Rather, we describe it. This ensures that the script can
            // continue working unmonitored even if we replace a device
            // or add more devices with the same features.
            //
            // The following source describes all the on-off buttons
            // in the living room.
            //
            // The condition is triggered if ANY of these buttons changes
            // state.
            "source": {
                // A `feature` represents something that a device driver
                // knows how to produce and/or consume.
                //
                // A feature itself is just a name, registered by one (or more)
                // device drivers. By convention, `switch/on-off` (which is
                // already registered in the standard library) is a feature
                // shared by all devices that have a on/off switch.
                //
                // The user interface can request a list of features currently
                // provided by drivers, and use it to populate the visual
                // scripting UX.
                //
                // Here, we could request a device driver that has several
                // features at once, but we only need one.
                "feature": "switch/on-off",

                // Tags are just strings, which may be attached/removed from the
                // user interface. By convention, `location/*` means a
                // human-readable description of where the device is placed.
                //
                // The user interface can request a list of tags currently
                // registered, and use it to populate the visual scripting UX.
                "tag": "location/Living Room",

                // By convention, `function/Light Switch` means that the user
                // intends to use the device as a light switch. After all, in
                // this script, we don't want to toggle the lights when the oven
                // is turned on.
                "tag": "function/Light Switch",
            },

            // Specify which feature we're actually using. This could be a subset
            // of the features mentioned in `source`.
            "feature": "switch/on-off",

            // We are not going to use the value provided by this `feature`,
            // so we don't need to bind it. We could just skip this key,
            // since we don't need it.
            "bind": null,

            // We are only interested in changes, not in specific value.
            // The device driver is in charge of checking this condition.
            // This `on change` means that whenver ANY of the sources
            // changes value, the condition is verified.
            "when": "on change",
        }
    ],

    // Every time the AND of conditions switches from false to true,
    // we execute the `execute` block.
    "execute": [
        {
            // `destination` behaves exactly like `source`.
            // We'll send a message to ALL devices that match
            // the criteria in `destination`.
            "destination": {
                // We only wish to talk to devices that are lights that
                // can be turned on/off.
                //
                // This feature is pre-defined in the standard library.
                // Its type is the enum "light/on" | "light/off" | "light/toggle".
                "feature": "light/on-off",

                // We only wish to talk to devices that are in the
                // living room.
                "tag": "location/Living Room",
            }

            "feature": "light/on-off",

            "instruction": "light/toggle",
        }
    ]
}
```

Similarly,

> If the entry door is locked and the motion detector notices motion, send an alarm to my SmartPhone.


```js
{
    "conditions": [
        // First condition: the entry door must be locked.
        {
            "source": {
                "feature": "door/locked-unlocked",
                "tag": "location/entry",
            },
            "feature": "door/locked-unlocked",
            "when": {
                // Equality check. Checked by the device driver.
                "Eq": "door/locked"
            },
        }
        // Second condition: the motion detector must notice motion.
        {
            "source": {
                "feature": "motion detector/motion-nomotion",
            }
            "feature": "motion detector/motion-nomotion",
            "when": {
                // Equality check. Checked by the device driver.
                "Eq": "motion detector/motion"
            }
        }
    ],

    // Every time the AND of conditions switches from false to true,
    // we execute the `execute` block. This means that we won't send
    // another message while the entry door is locked and there is
    // motion. However, if motion stops and resumes, we'll send
    // another message.
    "execute": [
        {
            "destination": {
                // All the devices that can be used to send a text message
                // to the user.
                "feature": "user agent/text message",
                // Restricted to the phone.
                "tag": "form factor/phone",
            }

            "feature": "user agent/text message",

            "instruction": {
                "message": "There's someone in your home!"
            },
        }
    ]
}
```

If you factor in the fact that:

- the scripts will continue working when we change the motion detector (including changing
    the brand of that motion detector), or the other devices;
- the scripts will resume working if there is a power outage;
- these scripts are easy to run in a power-efficient and bandwidth-efficient manner
    (assuming well-behaved device drivers).

these scripts are actually impressively concise for what they're doing. Also,
recall that most IoT applications these days are written with Turing-complete
programming languages, often with the highly dynamic but nearly impossible
to verify JavaScript language. By opposition, these tiny scripts could be
easily mechanically verified.

Let's look at a more sophisticated example before proceeding:

> If the entry door is locked and the motion detector notices motion in the living room, if I have a camera in the living room, send an picture to my SmartPhone.


```js
{
    "conditions": [
        // First condition: the entry door must be locked.
        {
            "source": {
                "feature": "door/locked-unlocked",
                "tag": "location/entry",
            },
            "feature": "door/locked-unlocked",
            "when": {
                "Eq": "door/locked"
            },
        }
        // Second condition: the motion detector must notice motion.
        {
            "source": {
                "feature": "motion detector/motion-nomotion",
                "tag": "location/Living Room",
            }
            "feature": "motion detector/motion-nomotion",
            "when": {
                "Eq": "motion detector/motion"
            }
        }
        // Third condition: if I have a camera in the living room
        {
            "source": {
                "feature": "camera/lazy-snapshot",
                "tag": "location/Loving Room",
            }

            // By convention, a lazy snapshot is a snapshot that is
            // only taken when all conditions are true (more precisely
            // when the `bind` is computed): we don't want to waste
            // the battery of the camera.
            //
            // The type of `camera/lazy-snapshot` is `image/*`.
            //
            // One could imagine having a feature `camera/lazy-snapshot-4k`
            // which prouces a `image/4k`.
            "feature": "camera/lazy-snapshot",

            // The snapshot is always "true".
            "when": "true",

            // Let's give a name to our snapshot.
            //
            // The semantics of `bind` mean that we need to have a value
            // to proceed. In particular, the condition can only ever
            // be validated if we have at least one camera in the living
            // room.
            "bind": "snapshot of the living room",
        }
    ],

    // Every time the AND of conditions switches from false to true,
    // we execute the `execute` block. Here, this could mean that
    // a camera in the living room has suddenly become available
    // while the entry is locked and the motion detector detects
    // movement.
    "execute": [
        {
            "destination": {
                // All the devices that can be used to send a photo
                // to the user.
                "feature": "user agent/image message",

                // Restricted to the phone.
                "tag": "form factor/phone",
            }

            // The type of `user agent/image message` is `image/*`.
            // We could imagine more restricted types that accept
            // only some image formats.
            "feature": "user agent/image message",

            "instruction": {
                "caption": "There's someone in your home!",
                "image": {
                    // Value captured by `bind` earlier.
                    // The type system ensures that both
                    // have the same type.
                    "$var": "snapshot of the living room"
                }
            },
        }
    ]
}
```

# Features and types

The first prototype of Thinkerbell didn't really have specific types,
just the usual number/boolean/string/... . We quickly
realized that this was a bad idea.

In your future SmartHome, everything is controllable by scripts and APIs.
This means that your lights, your water tap and your oven all can be turned
on/off, your heater, your oven, your fridge and your blowtorch (why not?)
all have a temperature setting -- and that confusing them can flood or
burn your house, killing you and your loved ones along the way.

Let me stress this: **in the SmartHome, types can be the difference between life and death**.

We followed the example of [F#'s unit-of-measure](https://www.semanticscholar.org/paper/Types-for-Units-of-Measure%3A-Theory-and-Practice-Kennedy/10652afefe9d6470e44f0d70cec80207eb243291) and introduced types
for temperatures and other physical measurements, On/Off
instead of simple booleans, etc. This was not sufficient
to solve the problem, because setting your freezer to
bath temperature is still a pretty bad idea.

Our solution was to differentiate *everything*. As can seen
above in the sample, we have values `light/on`, `light/off`,
`light/toggle`. These values are distinct from `oven/on`,
`oven/off`. We have a type of oven temperature and a different
type of freezer temperature and a different type of water
temperature. And, of course, temperatures and other physical
measurements had units-of-measure.

For instance, a device driver for freezers would register
(in pseudo syntax – this was done by an API):

```js
{
    "feature": {
        "key": "freezer/set-temperature",
        "type": "freezer/temperature",
    }
    "type": {
        "key": "freezer/temperature",
        "tuple": [
            "core/temperature"
        ]
    }
}
```

where predefined type `core/temperature` could itself handle ºC/ºF
conversion.

Two device drivers for freezers would conflict if they both
defined a feature `freezer/set-temperature` with a
structurally different type.

Once we have such definitions, implementing a type system for
this simple language is not complicated.

## Actual implementation

In the prototype implementation of Thinkerbell, types
were verified dynamically, during the execution of a script.

The real reason for this was that we operated under strict
deadlines and simply didn't have time to implement static
type checking, during the installation of a script.

There was also a fundamental issue that we would have needed
to resolve. Indeed, Thinkerbell supported dynamically removing
device drivers, which could unregister features/types dynamically.
This could have invalidated scripts that had already passed type-checking.
Worse, device drivers could then be re-added, with incompatible
feature/types definitions. We would have needed to specify a
semantics for the effect on previously-well-typed and
previously-ill-typed scripts.


# Surviving earthquakes, kind of

One of the key ideas behind Project Link is that the server needs
to live in the house of a end-user, who may have very little
technical knowledge, and it will live through famine and war.

Well, more precisely, a server would be unplugged because the
user would need to plug in the vacuum cleaner, or while it was moved
between rooms, and it would be rebooted during firmware
upgrades. All of this needed to happen unmonitored.

On the other side, old connected lightbulbs were bound to be
replaced by newer connected lightbulbs, cameras bought from
one company were bound to be replaced by cameras bought from
another company and scripts needed to continue working,
unmonitored.

So, we designed Thinkerbell and its lower-level API (one
can think of this API as the Thinkerbell VM) to
handle as much of this as possible.

We have seen above how Thinkerbell scripts (and its lower-level
API) access devices by their properties (features and tags),
rather than by unique identifiers. Features are exposed
by the device driver, while tags are entered by the user
whenever they install a new device or wish to reconfigure
it. This mechanism was designed specifically to
handle the problem of replacing devices by other devices,
possibly from incompatible vendor, without having to alter
their scripts (either Thinkerbell-level or remote scripts).

Now, the problem of server reboots is made easy to solve
by the simplicity of Thinkerbell scripts and the mechanism
of device drivers. Since Thinkerbell knows exactly which
conditions to monitor, and since it communicates with
the device drivers, Thinkerbell can store and persist the
information to disk - it is a simple boolean per condition.
If Thinkerbell discovers it has rebooted, it
can simply load the information from disk, request an
update-as-soon-as-convenient from device drivers,
check if conditions have changed, and proceed from there.


## Limitations

Persistence wasn't actually implemented in
Thinkerbell prototypes, due to lack of time.

The persistence mechanism discussed above doesn't handle the
case in which Thinkerbell is interrupted while executing
after conditions have become true. While several semantics
can be designed and implemented, we did not have the time
to consider and experiment on the topic.


# Process algebra (aka formalizing it)

The language was never fully formalized, largely because of the
deadline pressure mentioned above, and then because Project
Link pivoted suddenly.

However, the design of the language was made with the sketch
of a process algebra in mind. This clearly helped minimize the
language and design the type system. Also, the idea was to
eventually extend the language to:

- let scripts dynamically add or remove scripts;
- let scripts put themselves on temporary pause, e.g. to avoid spamming the user with messages;
- define pseudo-device drivers combining several tasks.

For all of this, having
a formal basis of the language would have
helped ensure that we knew what we were doing, in
terms of (non-)interference between scripts and device
drivers, infinite loops, etc.

Plus, it was fun to have.

Semi-formally, the syntax for the process algebra looked like:

```
// Rules
R, R' ::= 0
       |  R|R'       // Two parallel rules
       |  C => E

// Conditions
C, C' ::= 0
       | C && C'     // Expect two conditions
       | D when W(x) // Receive and bind

// Devices
D, D' ::= 0
       | D ∩ D' // Match two criterial
       | feature "some/name"
       | tag "some/name"

// When
W,W' ::= 0
       | W && W'
       | true
       | arbitrary string // interpreted by device driver

// Executions
E, E' ::= 0
       | E || E' // Execute two things in parallel
       | D<V>    // Send an instruction

// Values
V, V' ::= x // Variable
       | { key_1: V, key_2: V', ... } // Structured message
       | string
       | number
```

I never got around to writing a formal semantics for this
process algebra, but I suspect that it would be pretty easy.

# What now?

Well, this is the end of this post-mortem/brain dump.

I hope I have convinced you that Thinkerbell was pretty cool and some of the ideas
deserve a second life. If you're interested in doing so, have fun!

