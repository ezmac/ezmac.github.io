---
layout: post
title:  "Basic job control in shell"
date:   2018-06-08 11:38:37
categories: work_related shell
---

The other day, a friend of mine asked me "so, you know how you can do php -S localhost:1234, and then you hit something to stop it so you can put it in the BG?".  She was trying to run a server on her computer for development and wanted to run other commands in the same terminal.  The answer she wanted was ctrl+z, but the full answer is more complex.


Ctrl+z by it self only suspends a program.  When suspended, the process will be put into the T state, which means stopped by job control signal or because it is being traced.  The system continues to operate around it, but it does not receive cpu time.

{% highlight shell%}
sleep 100
^Z
ps aux|grep sleep
# tad      26006  0.0  0.0   7288   672 pts/50   T    08:32   0:00 sleep 100
{% endhighlight %}

The commands `bg` and `fg` are built in shell command that allow you to resume a job's execution and either put it in the background or foregrouns. Backgrounding is useful because it gives you access to your terminal again.  Foreground brings the task back into the current shell context, as though you didn't suspend it, and resumes execution.

In the example above, sleep will exist in the T state until it can resume running.  If the elapsed time has exceeded the sleep time, it will immediately exit when resumed.  Otherwise, it will continue sleeping until the elapsed time exceeds the sleep time.

A better example is something like ruby/node/php for a web server.  Since the original request was php, I'll use it as the example.
{% highlight shell %}
~/code/ezmac.github.io$ php -S localhost:1234
PHP 7.0.30-0ubuntu0.16.04.1 Development Server started at Fri Jun  8 09:29:22 2018
Listening on http://localhost:1234
Document root is /home/tad/code/ezmac.github.io
Press Ctrl-C to quit.
^Z
[1]  + 5066 suspended  php -S localhost:1234
{% endhighlight %}

While there is a server running and the port is allocated, the process receives no cpu time and will never respond.  If you visit localhost:1234 in a browser, you will see the connection established, but no data ever gets sent.

Running `bg` will put your process into the running state and your request will be answered.
{% highlight shell %}
~/code/ezmac.github.io$ bg
[1]  + 5066 continued  php -S localhost:1234
[Fri Jun  8 09:31:28 2018] 127.0.0.1:33270 [200]: /
[Fri Jun  8 09:31:28 2018] 127.0.0.1:33276 [404]: /%7B%7B%20site.url%20%7D%7D/images/%7B%7B%20post.image.feature%20%7D%7D - No such file or directory
~/code/ezmac.github.io$ [Fri Jun  8 09:31:29 2018] 127.0.0.1:33280 [200]: /favicon.ico
{% endhighlight %}

### Input/output

In shell, there are 3 main files that get used for tasks:

 - 0 - stdin (standard in)
 - 1 - stdout (standard out)
 - 2 - stderr (standard error)

When you start a task, it is given those files for it's own output and input.  When you background a task, it retains those outputs.  This means that your shell _and_ the process you backgrounded share an output.  When the background task prints some text, it gets inserted where your cursor was.  As seen above, it can get confusing when it inserts text where your cursor should be.

### Managing jobs

Your shell should come with a built in `jobs` command that allows you to see what jobs exist and their state.
{% highlight shell %}
~/code/ezmac.github.io$ jobs
[1]  - running    php -S localhost:1234
[2]  + suspended  vim README.md
{% endhighlight %}

If you're simply working with one job, bg and fg work as you'd expect.  If you have more than one, fg and bg work on the most recently used job.  To specify jobs, pass `%n` into bg or fg.  `fg %2`.  You can also use kill with `%n` to kill a process.  Some programs (like vim) don't respond well to being killed and will stay around until they've finished their output.

### Advanced backgrounding

If you want to skip the ctrl+z; bg step, you can suffix the command with a space and ampersand:
`php -S localhost:1234 &`

The space is not strictly necessary, but recommended for clarity.



### references
https://idea.popcount.org/2012-12-11-linux-process-states/


