It's no secret that all projects die, eventually, and that most projects die
before they have had a chance to have any impact. This is true of personal projects,
of open-source projects, of startup projects, of hardware projects, and more.

A few days ago, [the IoT project on which I was working](https://wiki.mozilla.org/Connected_Devices/Projects/Project_Lighthouse)
did just that. It was funded, it was open-source, it was open-hardware, and it
still died before having the opportunity to release anything useful. In our case,
we got pretty far, didn't run out of funds, and were a little polish away from
putting our first alpha-testers in front of a pretty advanced prototype before
a higher-level strategy pivot killed a number of projects at once.

Ah, well, risks of the trade.

Still, working on it, we managed to learn or come up with a few tricks
to increase the chances of surviving at least long enough to release a product.
I'll try to summarize and explain some of these tricks here. Hopefully, if you
have a project, regardless of funding and medium, you may find some of this
summary useful.

<!--more-->

# Death by Nobody-will-use-your-project

> tl;dr
>
> There are many good ways to test an idea way before the prototype is solid
> enough to show it up. Use them.

You have an idea. It's certainly a great idea. Your friends and your family
probably love it. A few geeks around you are enthusiastic, it can change the world!

At this stage, you are eager to start coding/designing/burning circuits/...
And that's great. By all means, if you're enjoying yourself, please carry on.
However, before you start committing to the project anything you might regret –
leaving your day job, spending your money or your investors' – you should make
sure that your project has a chance to gain users and traction.

If you come from a technological background, you may believe that you need to
put your project in front of users to get feedback, which means that you need
a pretty advanced prototype. *If you do this, you considerably increase your
chances of your project dying on you.*

So time to put your Survivor Hat.

There are many ways to get user feedback while the project is in its
infancy. Doing this used to be long, expensive and – just as important for an indie
project – boring. These days, there are a few services that do it for you
and actually bring money to your project in case of success. Yes, I am talking of [Kickstarter](http://kickstarter.com/), [Indiegogo](http://indiegogo.com/)
& [co](https://en.wikipedia.org/wiki/Comparison_of_crowdfunding_services). If you
believed that these crowdfunding platforms were about gathering money,
welcome to the worst-kept secret of the 2010s:
they are actually marketing platforms that will let you find out *if people are
willing to spend money on your project*. Which, in many cases, is a pretty good
measure of whether you should greenlight your own project. Just be sure to be
honest regarding the current state of your project and how the money will be
used. If you lie, even with the best of intentions, your funders will eventually
find out and they will be legitimately angry.

An alternative, depending on what you're doing, is to open a website and start
taking pre-orders (morally dubious), start letting interested parties subscribe
to an announcement mailing-list (better), or get creative. I have even seen weirder variants in
which non-technical founders started a fairly sophisticated online service
before they even hired a single developer – by hiring underpaid students to actually do
the work of the machine, at least long enough to determine whether users would
actually use them (neat trick, morally... weird). I have also seen at least one startup-turned-internet-giant who implemented only a subset of the promised
features and measured how many users clicked on buttons that were not
implemented yet (not a trick I like, but it worked for them).

Note that all these techniques help you determine your potential end users.
Unless your end users are developers, that's entirely orthogonal to getting
people to take part in the development of your project.

This list is clearly not exhaustive. Now that you have your Survivor Hat on,
you should be able to come up alternatives if you need to.

Regardless of the technique you use, spending a few days of work to measuring
your potential audience can save you from disastrous choices for your career
(or savings, or startup, ...). As a bonus, putting together some communication
will also serve to put your own thoughts in order, decide of your priorities,
get feedback from your future users. Also, if you need money, it can serve to
fund you (crowdfunding) and/or to convince investors.

# Death by Tests-need-users-need-tests

> tl;dr
>
> There are many good ways to test algorithms, UX, accessibility, ... before the
> prototype is solid enough to show it up. Use them.

At this stage, I'm assuming that you have identified your user population and
the conditions in which your project will be used. You will, of course, need
to adapt – you will need different testers for an OCR-for-the-blind or for a
superbowl-adblocker. For narrative purposes, though, I'll pretend that your testers
somehow lurk in your hallway.

So, your users are interested, your project is going full speed ahead. Great.
But, like it or not, full speed ahead is often still too slow with regard
to testing. Why? Because
just as code doesn't work until it has been {unit, integration}-tested, a
product doesn't work until it has been user-tested. Again, if you are thinking
as a developer, you will be tempted to wait until you have an advanced prototype.
But, if you are thinking as a developer, you also know that you don't wait until
the project is nearly complete to start adding {unit, integration} tests. This
is doubly true if your project involves hardware – just building the hardware
in the correct form factor can take months that you typically do not have.

So put on your Survivor Hat and find ways to run user tests
on a prototype that doesn't exist yet.

UX can be tested on mockups. If it's a GUI, a text interface or a braille
interface, draw it using any mockup tool, or
even on paper, grab some people in the hallway, see if they can guess how to
go from "I want to do this" to "I'm clicking the right button". If it's a
sound/voice UI, hide your mockup, grab some more people in the hallway, and
roleplay the speech recognition/text-to-speech engine, etc. Even wearable IoT
devices can be mocked up using wire, velcro, a smartphone and some bluetooth
accessories.

Similarly, you can user-test algorithms. Add a mockup UX on top of it – again,
it can be as simple as paper – grab some people in the hallway and run the
algorithm. If the algorithm is simple enough, you can again roleplay it. If it is
more complex, turn it into a command-line/REPL toplevel and run it behind the
scenes.

Let's go further. Are you building an IoT device? If you are lucky, you can
quickly rig together a prototype running your code on top of a smartphone,
smart tv, alexa or any existing device that your future users already own –
and if no smart device has all the capabilities you need for testing your code,
maybe a Bluetooth thingy can provide the missing features. Or perhaps you can
prototype your software application as a browser add-on. Or detach a single
feature and turn it into an App.
Suddenly, instead of having access to a handful of potential alpha-testers,
you can put your code in front of potential millions.

Again, regardless of the technique you use, spending a few weeks on mockups/early
prototypes will let you perform user test in parallel with the unit and integration
tests that you already run continuously. Much as unit/integration tests, user
tests will let you fix issues before they kill you. Hey, sometimes, [a prototype](https://vr.google.com/cardboard/)
can even *become* [a product](https://vr.google.com/daydream/).


# Now what?

Well, hopefully, this blog post gave you a few ideas. Now, put on your
Survivor Hat and start thinking about ways to make your project survive until
it has a chance to have an impact!

As a side-note, while I came up with the "Survivor Hat" as a narrative to
explain some of the techniques above to fellow developers and while we came up
with some of the ideas above mostly independently during Project Lighthouse,
this blog entry is largely about some of the "Lean Startup" concepts. So, if you
are interested you should find it easy to read more on the general idea.