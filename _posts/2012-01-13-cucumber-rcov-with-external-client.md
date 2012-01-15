---
layout: post
permalink: cucumber-rcov-with-external-client.html
title: Cucumber Test Coverage With an External Client
category: work
tags: github
---

I'm currently working on a Sinatra app that emulates a Subversion HTTP
server.  Much of the automated test suite is driven by Cucumber, which
sets up repository and checkout scenarios and exercises them using
various `svn` commands.  I recently wanted to get an idea of how well
these Cucumber tests exercised the application, and this is how I got it
working using rcov.

Cucumber does have rcov integration, but expects to use a driver like
Webrat or Capybara to handle HTTP requests.  In that environment
Cucumber is able to create an instance of your application and use mock
request and response objects to communicate between it and the driver.

Since I have cucumber calling out to `svn`, the application needs to be
listening for connections on a port that the svn client can connect to.
I previously had that working with a few changes to my `env.rb` file:

{% highlight ruby %}
`bin/rackup config.ru -D -P tmp/test_server.pid -p 5205`
sleep(2)

at_exit do
  Process.kill('KILL', File.read('tmp/test_server.pid').to_i)
end
{% endhighlight %}

The `-D` rackup parameter caused the app to start up in the background.
After giving the server a couple of seconds to start, there would be a
listener ready for the svn client to hit.  The `at_exit` block would
take care of shutting down the server when the test suite finished.
This wasn't very elegant, but it got the job done.  That is, until I
tried to introduce rcov.

rcov does its thing by loading a bunch of ruby code, tracking which
lines are executed until it exits, and then reporting on what was and
wasn't executed.  `bin/rackup` is loadable ruby code, so I was able to
make a small modification to get it working under rcov:

{% highlight ruby %}
`bin/rcov bin/rackup -- config.ru -D -P tmp/test_server.pid -p 5205`
{% endhighlight %}

I ran `rake features` and rcov happily wrote out coverage reports.
However, the reports were all wrong - barely any of the application code
was reported as covered, and I knew that much of it was definitely hit
during the test run.  The problem was the `-D` parameter; the code that
rcov was tracking loaded the application classes, fired up a detached
child to do the real work, and then quit.  Turns out the reports had
been written out before most of the tests had run.

Fixing this was pretty straightforward; rather that relying on rack to
run itself in the background, I moved that logic into `env.rb`:

{% highlight ruby %}
fork do
  exec 'bin/rcov bin/rackup -- config.ru -P tmp/test_server.pid -p 5205'
end
sleep(2)

at_exit do
  Process.kill('KILL', File.read('tmp/test_server.pid').to_i)
end
{% endhighlight %}

Success!  But there was something that bothered me a bit.  Cucumber has
better integration with rcov that I wasn't using.  Ideally, I would use
a rake task instead of my cucumber environment setup to control rcov:

{% highlight ruby %}
Cucumber::Rake::Task.new(:features) do |t|
  t.cucumber_opts = "--format pretty"
  t.rcov = true
end
{% endhighlight %}

I updated my rake task and changed the background rack app to run
without rcov, and again things ran but the coverage reports were way
off.  This time, rcov had measured coverage in the parent process where
all the cucumber things were happening and nothing in the child process
where the app was running
([rcov doesn't work with Kernel.fork](https://github.com/relevance/rcov/issues/77).)

There was no reason that the cucumber things needed to happen in the
parent, I just needed to switch things around so the rack app ran in the
parent process and the testing continued in the child.  This required a
bit of research as I needed the rack app to start in the current process
instead of a subshell.  Fortunately, rackup is ruby so I was able to
start the app in a more ruby-friendly way:

{% highlight ruby %}
Rack::Server.start(
  :app           => My::App.new,
  :environment   => 'none',
  :Port          => 5205,
)
{% endhighlight %}

Tests ran successfully but the output was full of Webrick debug and
access logs.  Passing some extra options to `Rack::Server.start` took
care of that:

{% highlight ruby %}
Rack::Server.start(
  :app           => My::App.new,
  :environment   => 'none',
  :Port          => 5205,
  :Logger        => Logger.new('/dev/null'),
  :AccessLog     => [],
)
{% endhighlight %}

This time the tests ran and looked great, but I didn't get any coverage
reports.  rcov was politely waiting for the call to Rack::Server.start
to return and it got SIGKILLed.  Further research turned up a couple of
interesting things: you can stop the server with a SIGINT, and the server
has callbacks.  I had never been happy with `sleep(2)` as a reliable way of
waiting for the server to start; in fact it had been responsible for a
few false negatives.

Since the rcov/server and tests were running in separate processes, I
needed a way for the callback in the server process to let the tester
process know it was OK to proceed.  I opted to use a pipe, where the
tester process would wait on a read and the server would write when the
callback triggered.

Also, the SIGINT from the child was now causing a return from
`Rack::Server.start` rather than killing the server process,
so I had to add an explicit exit to prevent the server process from
running all the tests again.

Now the rcov coverage reports were correct, and the server was starting
and stopping cleanly.  There was only one subtle problem left.  The exit
status of the parent process is used to indicate whether the test suite
passed, and it was always returning a 0 (pass) status even if a test
failed:

{% highlight bash %}
$ bin/cucumber features/log.feature
Scenario: Log                                   # features/empty_repo.feature:45
  When I run svn log in the root of my checkout # features/step_definitions/log.rb:13
  Then I get the error "you suck"               # features/step_definitions/repo.rb:331
    Expected the svn command to fail, but it succeeded (RuntimeError)
    ./features/step_definitions/repo.rb:332:in `/^I get the error "([^"]*)"$/'
    features/empty_repo.feature:47:in `Then I get the error "you suck"'

Failing Scenarios:
cucumber features/empty_repo.feature:45 # Scenario: Log

5 scenarios (1 failed, 4 passed)
29 steps (1 failed, 1 skipped, 27 passed)
0m12.935s

$ echo $?
0
{% endhighlight %}

This was a simple fix; the parent/server process just needed to
relay the child/tester exit status.  My final `env.rb` looks like this:

{% highlight ruby %}
rd, wr = IO.pipe
server_pid = $$
tester_pid = fork
if tester_pid.nil?
  wr.close
  rd.read
  rd.close

  at_exit do
    Process.kill('INT', server_pid)
  end
else
  rd.close

  Rack::Server.start(
    :app           => My::App.new,
    :environment   => 'none',
    :Port          => 5205,
    :Logger        => Logger.new('/dev/null'),
    :AccessLog     => [],
    :StartCallback => lambda { wr.write('started'); wr.close }
  )

  Process.waitpid(tester_pid, 0)
  exit $?.exitstatus
end
{% endhighlight %}

I'm pretty happy with the way this ended up.  I can use the existing
Cucumber/rcov integration and I learned some interesting details
about Rack servers.  Now to work on the missing coverage...
