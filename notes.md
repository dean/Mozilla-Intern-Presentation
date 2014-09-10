######## Switching from vendor libraries to peep ##########

Hi everyone! Im here to tell you about how we switched from using a vendor library
to peep! 

########## Who I am ############

First though, a little bit about myself. My name is Dean Johnson. 
I am a Senior in Computer Science, studying at Oregon State University.
I am interning with the SUMO team, which works on both support.mozilla.org
and input.mozilla.org.

So, how exactly did I switch over the way we manage our dependencies?

Lets take a step back to my first day. I come in to this huge exciting new
office at 8 in the morning, pick up my new laptop, go through the setup process
for new interns then met up with Mike, my mentor.

The first question I had was: so, how do I get started? Kitsune (support.mozilla.org)
has a few ways you can get started on the project. Their are installation instructions
for Windows, Linux, and OS X, but I wanted to install via Vagrant so I could
run it in an isolated environement. Personally, when running web apps I
like to do it from a linux environment, since thats the environment it will
be deployed in. When i started my intern project, we didnt have a vagrant
environment available for new developers.

############ Vagrant #############

So for my first project, I got to start working on setting up vagrant. 
For those of you unfamilair with vagrant, it is basically a wrapper around
a virtual machine provider for easy deployment of configurable virtual machines.

Setting up a vagrant box to automatically setup a dev environment for kitsune
helped give me a good introduction to the code base since I had to make sure
everything worked in the environment, and got me familiar with setting up redis
and elasticsearch.

Once I finally got a vagrant environment set up, I was excited. Finally,
I can start working on the webapp! Well, dreams were shattered when I tried
running the application and it took over 2 minutes to just run the default
manage.py runserver command. 2 minutes!!

After researching the issue, I realized that when we are loading all of the
vendor packages, we are iterating through all the paths listed in vendor/kitsune.pth
to find each package. Normally, this is not a huge deal, it is just some File IO
and is usually pretty quick. For example, my mentor was able to start the webapp
on his system in under 2 seconds, which is completely manageable. So why was
it taking so long for me to start my webapp?

########### Vagrant File I/O Stats ############

Knowing the only difference between my mentors environment and mine were that
I was running in a VM, I started looking into performance issues with
VirtualBox. As it turns out, when you have files shared between your local filesystem
and your VM, access to those files gets REALLY slow. And to make it worse, the
performance for shared file systems is _worst_ on OS X. 

If you look at the slides, the purple box is Virtualbox shared folders. This
is what was being used in the Vagrant environment that caused it to be so slow.

I was able to hack my way around some of the timing issues by munging the way the
application sets up the path to access vendored libraries, but still was looking at
~ 1 minute load times.

Next, I figured I could deal with our web app loading slowly, so I tried working on a bug.
I was originally under the impression that loading
the webapp was what was taking a long time, and reloads would not take long at all.
I was wrong. Working on that bug, I learned how much of an impact
the tools you use for development can have on developer productivity.

After finishing up that bug, I realized how unproductive
I was being at the cost of a software problem that could be fixed, so I started
looking into ways of making everything load even faster.

Some of the things I did to speed up my environemnt in the meantime were:
tarball all of our packages and attempt to pip install them. Doing this I was
able to get load times down to ~30 seconds.

Talking with the team, we discussed different options to speed up the vagrant
environment, but ultimately we decided on peep. As it turns out, its pretty awesome,
and we already had a bug filed for it on bugzilla. 

############ Peep ##############

Now, for those of you who dont know what peep is, its a great tool.
It was created by Erik Rose and is basically a wrapper around pip that
ensures each package you are installing is the exact package you are looking
for.

* Slide with # sha256: wklqjelkqwjekljewq
             requests==0.9.0

What peep does is when its asked to install a package, it gets the hash of the
packages tarball, then compares it to the hash you have put down in your
requirements files. This has the same effect as vendoring your libraries
since it ensures that the contents of all your packages are the same as
when you first recorded them.

So, what does this mean for our system?

############# Previous Kitsune Org #################

This is how our repo used to be set up:

    kitsune
        manage.py
        ...
        /requirements
            compiled.txt
        /vendor
            /src
            /packages
            /tarballs
            kitsune.pth

Because I didnt manually want to go through each package and find its hash
then add it to the requirements file, I wrote a tool to do it for me. 
Its called peepify! What you do with it is point it at the top level directory
of a project and it will inspect the .gitsubmodules file and use it to
download tarballs for all of your git submodules, hash them, then create a
requirements file based off those tarballs hashes and urls.

To take care of the /packages directory, I used a tool willkg wrote called poopify.
That tool looks at each package in your /packages directory and looks at the top
level directory for a PKG-INFO file. If it finds it, it opens the file and finds
the version of the package we are using, then puts it into a requirements file.


################ Now Kitsune Org ##################


Which changed our structure into the following:

    kitsune
        manage.py
        ...
        /requirements
            compiled.txt
            git.txt
            pypi.txt
        /vendor
            /tarballs

The vendor/src was turned into a peep-compatible file named git.txt, and
vendor/packages into pypi.txt. 

So after finishing switching our vendor system to to peep, I now had to worry
about deployment. Because we couldnt completely emulate production/staging in a
VM (we simply dont have all the scripts) we had to go through a tedious process
of:

############# Deploy Attempt [1] ##################

1. Fix error
2. Commit
3. Push
4. Deploy
5. Probably go back to 1

After doing this for a while, we decided to write up a test plan to send to
the team, so we had a more concrete plan of what we were doing.

"
Bugzilla: https://bugzilla.mozilla.org/show_bug.cgi?id=1035319
Github PR: https://github.com/mozilla/kitsune/pull/2083

# Overview
* Whats been done
"

########### Attempt [2] ###############

What we decided on was:
* Testing the new virtualenv
* Testing the new/modified packages
* Testing other important features

"
# Whats Been Done
* peep.py added to /scripts. This will be our new method of deployment. For more information see the peep documentation. [1]
* /vendor/src removed and package requirements saved in requirements/requirements_src.txt
    * These packages are exact commits we had checked into /vendor/src except for django-cronjobs, statsd, django-product-details, and jingo-minify.
* /vendor/packages removed and package requirements saved in requirements/requirements_packages.txt
    * Each package version is an approximation taken from its PKG_INFO file.
* Pillow added as a requirement to requirements/compiled.txt
* All Travis scripts are updated.
    * Build passes all tests. [2]
"

############### Virtualenv testing ######################

To test out the new virtualenv our process basically went:
1. Deploy to stage
2. Test that the site works (loads) and cron jobs run correctly.
    * Cron jobs had issues in the past due to path issues with the virtualenv
      using system level packages instead of the packages from the virtualenv.


############## Testing new/modified packages #################

When I was using peepify to switch over our dependencies from vendor to
peep, I ended up running into some issues installing the new packages.
As it turns out, 4 of the packages we had in our gitsubmodules were
incorrectly setup to be installed via pip, which as I mentioned before is
wrapped around by peep, so we had to change the commit that we referred to.
In some of the cases the pip issues were fixed in previous commits, in others
I had to go fix them myself.

We test:
    Graphite - A tool we use to monitor metrics for various events on the site
    Check the styling on the site

"
# Testing the new/modified packages
* All changes made to the following packages were to commits that fixed packaging issues with pip. Sometimes, this included upgrading versions as well.
django-cronjobs
* Changes: New updates and fixed packaging issues.
* Check to make sure collect_tweets can run properly.
pystatsd
* Changes: New updates and fixed packaging issues.
* Check Graphite for login events, which should be triggered via tests.
jingo-minify
* Changes: New updates and fixed packaging issues.
* Make sure styles are correct and we are bundling js/css files.
    * The git url for this in requirements/requirements_src.txt needs to be updated once the PR [3] is merged.
django-product-details
* Changes: Fixed packaging issues.
* Nothing changed other than the packaging issue, so nothing should go wrong here.
"
"
# Testing other important features
"

For the last part of the test plan, I put down the important pieces of the
site to test, just to be sure nothing was broken.


################## Testing important features ###################

These were:
* Test Search
    * Searching questions
    * Searching documents
    * Checking filters work properly during searches
* Test Forums/Questions
    * Creation of new questions
    * Responding to questions
    * Viewing questions on forums
* Test Documents/Help Articles
    * Editing documents
    * Viewing documents
* Test AoA
    * Look for recent tweets and connect a Twitter account.
* Test Localization
    * Dashboard loads and shows results nearly identical to production.

* Lastly, run QA tests to make sure nothing else is broken.

After finishing all of these, we left the branch on staging for the weekend
to make sure nothing broke, intending on deploying it to production TODAY.
Like, literally later today. There is a couple more kinks to work out before
I am confident in saying it is ready to go, but it is just about there.


################# What did I learn? ###################

What did I learn?
* Developer tools are incredible and spending time to make your development
  environment fast, is almost always worth the time.
* A LOT about pip, virtualenv, and how they work internally.
* AngularJS, ElasticSearch, Django


############### Thank yous ################

Thanks:
* Mike Cooper (mythmon) - Mentor
* Ricky, Rehan, Topal - SUMO Team
* 2nd floor kitchen
* Tyler, Ian, Thomas - Roommates
* Jill, Misty, Kimber
* Other interns

############## Questions ##############

Questions?
