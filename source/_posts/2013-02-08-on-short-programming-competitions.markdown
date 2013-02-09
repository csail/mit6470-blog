---
layout: post
title: "On Short Programming Competitions"
date: 2013-02-08T23:07:32-05:00
comments: true
external-url:
author: "Victor Costan"
categories:
---

In November 2012, 6.470 held its first short programming competition. We
challenged students to use their CSS, JavaScript and SQL skills to solve as
many problems as possible in a short interval of time. Students submitted their
solutions as soon as they finished them and automatically received grades in
real time. This post focuses on the infrastructure that we put together to make
that happen.


## Web Services FTW

The most important decision that we took was to separate the front-end from the
back-ends that graded the students' submissions. We have different backgrounds
and programming tastes, so agreeing on a
[HTTP API between the front-end and the back-ends](https://github.com/csail/mit6470-hackathon#grading-endpoints)
allowed each of us to use to our favorite tools.

We decided to make the back-ends stateless and synchronous, and push all the
authentication, data storage and and scheduling complexity into the front-end.
This way, we didn't end up duplicating boilerplate code, and our back-ends were
easy to test. For example, the JavaScript and CSS back-end has a
[bare-bones submission form](https://github.com/csail/mit6470-grader-sample/blob/master/views/grdr/form.erb)
that we used to test the problem graders.

In the end, we used [Ruby on Rails](http://rubyonrails.org/) to build the
student-facing front-end. The grading back-end for SQL was built on PHP and
deployed on an [Amazon EC2](http://aws.amazon.com/ec2/) instance. The grading
back-end for CSS and JavaScript used [Sinatra](http://www.sinatrarb.com/) and
was deployed on a server in MIT CSAIL.


## The Front-End

Our front-end authenticates students, shows the current round's problems,
collects students submissions, associates them with the scores emitted by the
grading back-ends, and puts together the score board. It also has an
administrative interface that let the staff monitor submissions for
irregularities and change grades in response to disputes.

Ruby on Rails is heavier than other frameworks, but it is ideal for building
Web applications with complex business logic. We made heavy use of the
[activeadmin gem](http://activeadmin.info/) for all the staff-facing admin
pages.

My biggest regret was that we did not figure out
[zero-downtime deployments](https://ariejan.net/2011/09/14/lighting-fast-zero-downtime-deployments-with-git-capistrano-nginx-and-unicorn)
for the front-end. We took agile development to the extreme, and made
[quite a few changes](https://github.com/csail/mit6470-hackathon/compare/e0e1706197ac19b23a8ae98224b9227c05e4bffb...6173693123dd17b64274392c476a2e3b3fb682ad)
during the actual contest. We could not push to production during the three
30-minute rounds, because restarting the application server would have
interfered with the grading of student submissions, so we had three 10-minute
windows to deploy changes.

We open-sourced our front-end. If you want to run your own competition, head
over to the
[project's GitHub page](https://github.com/csail/mit6470-hackathon/).


## Grading CSS and JavaScript

These days, most Web applications use [phantom.js](http://phantomjs.org/) or
[Selenium WebDriver](http://seleniumhq.org/) for integration testing. However,
these tools are designed for applications whose source code is not hostile to
the test code. In a contest, we have to assume that enterprising students will
try to detect and tamper with the test fixtures. We also wanted to perfectly
replicate the browser environment, so we didn't like the [missing features in
phantom.js](https://github.com/ariya/phantomjs/wiki/Supported-Web-Standards).

These were the major issues on our minds when we wrote the grading back-ends.

* attacks on the grading code that change a submission's score
* attacks that break grading for future submissions
* attacks that leak secrets such as test data and the back-end's URL

We ended up taking advantage of
[Chrome's remote debugging feature](https://www.webkit.org/blog/1875/announcing-remote-debugging-protocol-v1-0/),
which is the API used by the
[Chrome Development Tools](https://developers.google.com/chrome-developer-tools/).
When we needed to run JavaScript, we used the
[Runtime.evaluate](https://developers.google.com/chrome-developer-tools/docs/protocol/tot/runtime#command-evaluate)
interface, so submissions could not use DOM inspection or object-space walking
to find our code or tamper with it.

When grading a submission, the back-end started an
[Xvfb](http://en.wikipedia.org/wiki/Xvfb) virtual X server and a Google Chrome
instance configured to use a
[temporary user data directory](http://www.chromium.org/user-experience/user-data-directory).
The back-end killed the processes that it started when the test was over. This
prevented submissions from storing rogue state using
[localStorage](https://developer.mozilla.org/en-US/docs/DOM/Storage),
[IndexedDB](https://developer.mozilla.org/en-US/docs/IndexedDB), or other
Chrome-specific APIs that we weren't aware of. Xvfb gave us
[full control over the X screen resolution](http://www.rubydoc.info/github/pwnall/webkit_remote/WebkitRemote/Process:initialize),
which helped us write robust tests. Per-submission X sessions ensured that
submissions cannot store rogue state using the X server. We also
[disabled some Chrome features](https://github.com/pwnall/webkit_remote/blob/fe4da92b7e9bb9322d523ecfcf050c6e6b4010d5/lib/webkit_remote/process.rb#L138)
to reduce the attack surface and improve test consistency.

The grading back-end provided the students' submissions to Chrome using a
separate server configured to only accept connections from `127.0.0.1`. This
way, submissions could not use `window.location` to get the grading endpoint's
URL. Unfortunately, students could still learn the backend server's IP address
by causing Chrome to make a request to their custom server. This can be easily
achieved using the `background-url` directive in CSS, or the `Image`
constructor in JavaScript. In a future revision, we will
[set a custom PAC script](http://peter.sh/experiments/chromium-command-line-switches/#proxy-pac-url)
to restrict network access to a white-list of hosts, which will most likely
consist of `127.0.0.1` and `cdnjs.com`.

The grading back-end contains a domain-specific language that let us express
each problem's testing strategy in a concise manner, and separates the
problem specifics from the set-up code that is common to all problems. For
example, [this problem](https://github.com/csail/mit6470-grader-sample/blob/14298b5ac2359718383249d767c34baa3a01d80f/problems/css_sizing.md)
is tested using
[this code](https://github.com/csail/mit6470-grader-sample/blob/14298b5ac2359718383249d767c34baa3a01d80f/problems/css_sizing.rb)
and
[this page template](https://github.com/csail/mit6470-grader-sample/blob/14298b5ac2359718383249d767c34baa3a01d80f/problems/css_sizing.html.erb).

We open-sourced the JavaScript and CSS grading back-end, and simplified
versions of some of the contest's problems. The
[GitHub page](https://github.com/csail/mit6470-grader-sample) has the code and
instructions for setting up your own instance.


## Conclusion

Arctic Heist 2012, our first mini-programming competition, was a blast. The
student excitementand positive feedback made all our efforts worth it. In this
post, we shared some of our code and technical insights that made the contest
happen. We hope that these will be useful to other contest organizers.
