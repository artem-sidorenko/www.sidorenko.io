+++
date = "2015-08-24T07:56:54+02:00"
tags = [ "froscon", "conference" ]
title = "FrOSCon 2015 impressions"
+++

[FrOSCon](http://froscon.org) is a conference about Free Software and Open Source in Sankt-Augustin near Bonn, Germany.
Here I want to give my impressions from the visit this year. (btw, it was the 10th FrOSCon this year)

<!--more-->

# Day 1

<a href="voucher.jpg"><img style="float: right; margin-left: 1.5em;" src="voucher.jpg" width="150" title="Coffee voucher" ></a>
<a href="entrance.jpg"><img style="float: right; margin-left: 1.5em;" src="entrance.jpg" width="200" title="Entrance queue" ></a>

It took about 15 mins to get in, so I missed the start of the first talk. In the conference bag was the usual set of conference program, coffee voucher and flyers from sponsors.
I was really suprised to see so much "culture" talks in the program, usually such talks will be hold on Agile conferences. I joined directly the first talk from sipgate about OpenSpace and OpenFriday.

## Thank God it's Open Friday

[Corinna Baldauf](http://finding-marbles.com/), Scrum Master from sipgate, talked about their experience over last 4 years @sipgate with Agile methods and problems, discussions with C-level about trusting the employees.

One very interesting idea was to have a slack time between the sprint changes(NDS day = Nach Dem Sprint Tag): team members can do anything they want (solving issues, maintenance, research etc). Well, it wasn't really transparent and didn't really solve the issues with "technical dept".

After some further tries they implemented a regular "Open Friday" with the Open Space format. This turned to be a very good idea and was later implemented for entire sipgate company. So, its more or less like a "Fed-Ex" day, but on the regular base. This helped to get transparency, provided an efficient format and enabled knowledge transfer, innovation across the entire company. Very interesting experience:)


[**[Slides]**](https://docs.google.com/presentation/d/1SSxU4GH98jkgKO0_ToXbmeLl_EhL2h8LY3_8oDsEcUU/edit?pli=1#slide=id.p)
[**[Report]**](http://agilealliance.org/files/6514/3751/8832/ExperienceReport.2015.Bauldauf.Thank_God_its_Open_Friday.pdf)
[**[Video]**](http://media.ccc.de/browse/conferences/froscon/2015/froscon2015-1600-thank_god_it_s_open_friday.html)

## How To Get Your Patch Accepted

<a href="https://twitter.com/lhawthorn/status/635028245872816128"><img style="float: right; margin-left: 1.5em;" src="patch2.jpg" width="200" title="Superpower of reviewers" ></a>
<a href="patch1.jpg"><img style="float: right; margin-left: 1.5em;" src="patch1.jpg" width="150" title="Why its important to provide good commits?" ></a>

[Lorna Mitchell](http://www.lornajane.net/) talked about review criteria, quality of submitted patches and in total provided a good view from code maintainer or reviewer perspective.

Key messages:

- "3 ingredients: code, words, perspective"
- Invent acceptance criteria for your patches
- Avoid any commented code in your codebase
- Follow the contribution guides and create own for your projects
- Ask yourself how you can break your code as the end-user will do it for sure
- Create tests
- Follow coding standards
- Provide good commit messages: they will help to understand the reasons for particilar code after some years
- Learn to use your tools (e.g. git): squashing of commits etc
- No applogies in submissions: either the code is good or it needs to be better
- Use automated builds/testing
- Rebase/merge your code on the latest master and fix the issues
- A pull/merge request is a first step in the conversation about your change
  - Describe the use-cases and intention in your pull/merge requests
  - Cover the question "why a maintainer should merge this?"
  - Be prepared to invest time for improvement of your pull/merge requests
- Do code reviews of foreign code to learn and improve your coding style and quality
- Provide documentation for your code, comments in the code as well
- Reject the contributions youself if code is bad or something is missing

[**[Slides]**](https://speakerdeck.com/lornajane/get-your-patch-accepted)
[**[Video]**](https://media.ccc.de/browse/conferences/froscon/2015/froscon2015-1622-how_to_get_your_patch_accepted.html)

## Minix 3

<a href="https://twitter.com/nerdbabe/status/635043352954707968"><img style="float: right; margin-left: 1.5em;" src="minix2.jpg" width="150" title="Andrew S. Tanenbaum on stage" ></a>
<a href="https://twitter.com/TimKrajcar/status/635043037580795908"><img style="float: right; margin-left: 1.5em;" src="minix1.jpg" width="150" title="Packed house for Andrew Tanenbaum" ></a>

[Andrew Tanenbaum](https://en.wikipedia.org/wiki/Andrew_Tanenbaum) talked about the usual problems with operating systems and software and gived an overview over Minix evolution in the last years. [Minix](http://www.minix3.org/) is a free open-source operating system designed to be highly reliable(more reliable than Windows and even Linux).
<br/><br/><br/>

<a href="minix4.jpg"><img style="float: left; margin-right: 1.5em;" src="minix4.jpg" width="150" title="The Computer Model (Windows edition)" ></a>
<a href="minix5.jpg"><img style="float: left; margin-right: 1.5em;" src="minix5.jpg" width="150" title="I admit I was wrong" ></a>

From my POV Minix isn't a research/example OS only anymore, like it was in the past. It runs the NetBSD packages, can be installed on the Boards like BeagleBone and might be very interesting for embedded development. There are interesting ideas like live OS upgrade without affecting the running processes. Such thing can't really be realized the same way on the other operating systems.

[**[Video]**](https://media.ccc.de/browse/conferences/froscon/2015/froscon2015-1647-minix_3.html)

## Go Away Or I Will Replace You With A Very Little Shell Script

<a href="https://twitter.com/nerdbabe/status/635088127569358849"><img style="float: right; margin-left: 1.5em;" src="shell2.jpg" width="150" title="so why is computer science so damn hard?" ></a>
<a href="https://twitter.com/nerdbabe/status/635088127569358849"><img style="float: right; margin-left: 1.5em;" src="shell1.jpg" width="150" title="so why is computer science so damn hard?" ></a>

[Kristian Köhntopp](https://twitter.com/isotopp) gives some of his views on DevOps, culture, development of culture. There were quite interesting examples from his professional experience. Thoughts with real examples in direction like "You don't want to have issues in production? Then break you production reguary by youself! Do your tests in production!" are very interesting, espessially when you see something like "Budgeting downtime"

<a href="https://twitter.com/nerdbabe/status/635086302598316032"><img style="float: right; margin-left: 1.5em;" src="shell4.jpg" width="150" title="feature development vs infastructure development" ></a>
<a href="https://twitter.com/nerdbabe/status/635088127569358849"><img style="float: right; margin-left: 1.5em;" src="shell3.jpg" width="150" title="so why is computer science so damn hard?" ></a>

[**[Slides]**](https://plus.google.com/+KristianK%C3%B6hntopp/posts/7D4SDo1wgjM)
[**[Video]**](https://media.ccc.de/browse/conferences/froscon/2015/froscon2015-1500-go_away_or_i_will_replace_you_with_a_very_little_shell_script.html)

<br/><br/>

# Day 2

The second day started exactly like the first one: with a huge queue to get coffee.

<a href="https://twitter.com/nextlevelDE/status/635382849550569472"><img src="coffee1.jpg" width="150" title="... Start mit Kaffee, natürlich." ></a>

## How to Motivate Your Developers

<a href="devs2.jpg"><img style="float: left; margin-right: 1.5em;" src="devs2.jpg" width="150" title="About motivation" ></a>
<a href="https://twitter.com/sjstoelting/status/635390841712979968/photo/1"><img style="float: left; margin-right: 1.5em;" src="devs1.jpg" width="150" title="How to Motivate Your Developers - Thinks to avoid" ></a>


[Anna Filina](http://annafilina.com/) talked about developers motivation. She tried to take different perspectives (e.g. from developer and management) and to provide ideas to management how to reach the higher motivation and cover (even possible unknown) needs and wishes.

This also included different views on experts and beginners and even different motivation points were covered.

[**[Slides]**](https://speakerdeck.com/afilina/how-to-motivate-your-developers-2)
[**[Video]**](https://media.ccc.de/browse/conferences/froscon/2015/froscon2015-1575-how_to_motivate_your_developers.html)

<br/><br/><br/><br/>

## Beginning of the End or End of the Beginning?

[maddog](https://en.wikipedia.org/wiki/Jon_Hall_%28programmer%29) gived a keynote. There were a lots of interesting (insider) stories from the history of computers and his view on the future. Very interesting and funny:)

[**[Video]**](https://media.ccc.de/browse/conferences/froscon/2015/froscon2015-1640-beginning_of_the_end_or_end_of_the_beginning.html)

<a href="https://twitter.com/dataintransit/status/635405233947037696"><img style="float: left; margin-right: 1.5em;" src="maddog1.jpg" width="150" title="Now: Difference Engine & Ada Lovelace"></a>
<a href="https://twitter.com/lornajane/status/635405609651830784"><img style="float: left; margin-right: 1.5em;" src="maddog2.jpg" width="150" title="Having my education completed by Mr John 'maddog' Hall"></a>
<a href="https://twitter.com/dataintransit/status/635404858556854272"><img style="float: left; margin-right: 1.5em;" src="maddog3.jpg" width="150" title="going waaaaay back into history"></a>
<a href="https://twitter.com/tutnix/status/635400644405428225"><img style="float: left; margin-right: 1.5em;" src="maddog4.jpg" width="150" title="Lunchtime is nearly over at froscon"></a>

<br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/>

## Der Linux Netzwerk Stack

[Florian Westphal](http://www.strlen.de/) explained the architecture of network stack inside of Linux kernel and gived a good overview over the flow of bits&bytes there. The backgrounds and reasons were quite interesting, now I know who is `ksoftirqd` and why support of fast cards (e.g. 10Gbit) is a bit complicated.

[**[Slides]**](http://www.strlen.de/talks/linux_netzwerk.pdf)
[**[Video]**](https://media.ccc.de/browse/conferences/froscon/2015/froscon2015-1678-der_linux_netzwerk_stack.html)

## Docker in Produktion

Last FrOSCon talk and the room was quite full.  Its clear - its a topic like Docker.

[Sebastian Mancke](https://github.com/smancke) from [Tarent](http://www.tarent.de) shared the experience on Docker in production. They built the full automation chain for deployment with Docker:

<a href="https://twitter.com/dataintransit/status/635405233947037696"><img style="float: right; margin-left: 1.5em;" src="docker1.jpg" width="150" title="Docker in Production"></a>

- every change must to be packaged to the image
- this image gets tested
- and shipped to production

In the same time, all used solutions must be as simple and small as possible. No unneeded complexity!

For me it was the first Docker talk at all, which covered my questions and expectation on the productive usage, requirements and architecture.
The guys wrote their own orchestrator [gig](https://github.com/smancke/gig), which covers "missing" features from fig/docker-compose, but aims to be compatible with fig/docker-compose YAML files. The mix and implementation of environment parameters for images is just great!

[**[Slides]**](https://github.com/smancke/talks/tree/master/2015_froscon_docker_in_production)
[**[Video]**](https://media.ccc.de/browse/conferences/froscon/2015/froscon2015-1596-docker_in_produktion.html)

# Other stuff

<a href="https://twitter.com/_raphaelj/status/635054652749410304"><img style="float: left; margin-right: 1.5em;" src="other1.jpg" width="150" title="Schaut vorbei und versucht den Trockner via Schnittstelle zu stoppen :)"></a>
<a href="https://twitter.com/dataintransit/status/635034840992219136"><img style="float: left; margin-right: 1.5em;" src="other5.jpg" width="150" title="Die @teckids_eV übernehmen den Stand. Hallo Nachwuchs!"></a>
<a href="https://twitter.com/ElectricMaxxx/status/635361744479744000"><img style="float: left; margin-right: 1.5em;" src="other6.jpg" width="150" title="Nächstes Jahr kommen auch meine Kids mit und natürlich mein Schatz Froscon ist wie eine große Familie"></a>

<a href="https://twitter.com/_DrBunsen_/status/635417500948262912"><img style="float: left; margin-right: 1.5em;" src="other2.jpg" width="150" title="That escalated quickly..."></a>
<a href="https://twitter.com/_DrBunsen_/status/635417500948262912"><img style="float: left; margin-right: 1.5em;" src="other3.jpg" width="150" title="That escalated quickly..."></a>
<a href="https://twitter.com/_DrBunsen_/status/635417500948262912"><img style="float: left; margin-right: 1.5em;" src="other4.jpg" width="150" title="That escalated quickly..."></a>
