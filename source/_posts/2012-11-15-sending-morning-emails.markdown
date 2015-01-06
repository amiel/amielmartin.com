---
layout: post
title: "Sending Morning Emails"
date: 2012-11-15
updated: 2015-01-05
date_formatted: November 15, 2012
updated_formatted: Janurary 1, 2015
comments: true
categories:
---


NOTE: I originally wrote this when I was still working on [Stride](http://strideapp.com). I've reposted it here for posterity. Here is the [original post](https://web.archive.org/web/20130522081001/https://strideapp.com/blog/2012/11/sending-morning-emails/).

Reliably delivering time-based emails (daily, weekly, etc.) to users has always been a little tricky for web-based applications. There are multiple possible failure points, and we need to contend with different time zones.

Currently, Stride sends two time-based emails:

1. Your Monday morning weekly recap
2. Task reminders

We wanted both of these emails to arrive in the user's inbox in the morning. At first, this seems like a simple specification; we'll just send the emails off at 7am.

But hold on a minute — if we send the Monday morning email at 7am PST, users in New York won't get their email until 10am, and even worse, our users in Australia won't get it until 1am on Tuesday; that certainly isn't Monday morning. What if we send it early enough so that everyone gets it before Monday morning? Unfortunately, if we send the it at 7am in eastern Australia, our users in Hawaii will get their Monday morning email at 11am on Sunday. That just isn't going to cut it.

We decided to batch up the emails and send them off depending on each user's time zone. Here's how it works:

<!--More-->

### Getting the time zone from users

The only way to ensure people get their emails in the morning for them is to know their time zone, and while it's important to us that people get their emails in the morning, we didn't want to force them to configure a time zone. Don't get me wrong: if we hear that our users want the option to set their time zone, we'll provide it, but we want to keep the interface as simple as possible. You can read more about this decision in [Nathan's article](http://blog.strideapp.com/2012/09/the-invisible-interface/).

Your browser knows what your time zone offset is, and your daylight savings configuration. So, thanks to [jstimezonedetect](https://bitbucket.org/pellepim/jstimezonedetect), we can make a pretty good guess as to your time zone setting using JavaScript.

![Diagram of timezone detection](/images/autodetect_tz.png)
{% img /images/autodetect_tz.png Diagram of timezone detection %}

``` javascript Using jstz jstimezonedetect and jQuery to send the timezone to the server
var timezone = jstz.determine_timezone();
var timezone_name = timezone.name();

$.ajaxPrefilter(function(options, originalOptions, xhr) {
  xhr.setRequestHeader('X-Timezone', timezone_name);
});
```

We can save the users "automatic" time zone like this (the rest of the code samples are written for Ruby on Rails, but the concepts should apply to any web-based environment):

``` ruby Storing user timezones; This can be placed in a before_filter
js_timezone = request.env['HTTP_X_TIMEZONE']
if js_timezone.present? && UserTimezone::TIMEZONE_TZNAMES.include?(js_timezone)
  current_user.update_attributes auto_timezone: js_timezone
end
```

In case we ever need to overwrite the automatic configuration for any reason, we have a separate column in the database that allows us to do so.

### Determining when to send the emails

Now that we have most user's time zones, we can send them their Monday morning and task reminder emails at 7am in their time zone. Here's how that works.

We run a cron task every hour. Using the Monday mailer as an example, the first thing it does is answer the following question: "In what time zones is it now 7am on Monday?" Here is that question in code:

``` ruby
TIMEZONES = ActiveSupport::TimeZone.all

def self.timezones_where_the_day_and_hour_are(wday, hour, time = Time.current)
  TIMEZONES.select { |z|
    t = time.in_time_zone(z)
    t.wday == wday && t.hour == hour
  }.map(&:tzinfo).map(&:name)
end
```

Confusingly enough, the answer could be zero time zones, or quite a few. And, of course because of Daylight Savings Time, the answer will be different depending on the season.
Armed with the list of time zones in which it is time to send "morning" emails, we can make a quick indexed query for users in those time zones:

``` ruby
# Given a list of timezones, return all the ids for users in all those timezones.
def self.user_ids_in_timezones(timezones)
  return [] if timezones.empty?

  timezones << nil if timezones.include?('Etc/UTC')
  where(timezone: timezones).pluck(:user_id)
end
```

Users without a time zone configured will get their email at 7am UTC.

In the case of task reminders, we store the time zone and date on each reminder so that we can do an indexed query given the appropriate time zones on any given date.

### Queueing up the emails

Now that we have a list of users that need Monday morning emails (or task reminders that need to be sent), we can go ahead and fire off those emails. In order to track the progress and hopeful success of each email, we queue each email individually with [Resque](https://github.com/blog/542-introducing-resque), a background job runner written by the awesome folks at Github.

``` ruby Queueing up emails
UserTimezone.user_ids_in_timezones(timezones).each do |user_id|
  puts "Queuing the MondayMailer to #{ user_id } at #{ time }!"
  Resque.enqueue(::MondayMailer, user_id)
end
```

Not only is Resque great run to processes in a Rails environment, it also has a front-end to inspect the jobs in the queue, what's currently running, and retry failed jobs.

### On Cron and Queues

Another part of this system that has been handy for us is the way we run cron jobs.

We've had a lot of issues in the past running cron jobs for Rails. Logging is difficult, debugging failures is hard, and the environment is tricky to set up. So instead of directly running Rails code, we have cron just queue up a resque job. This simplifies the environment our cron task needs and moves the logging and failure handling to resque, which is far more desirable.

The Resque job takes a time for when the job was requested. This way, the queue could be backed up (or have failed entirely) so while jobs might still run late, they will at least know what time they were meant to be run.

Here's the script that cron calls directly (script/rescque_cron_task):

``` ruby
#!/usr/bin/env ruby

require 'resque'

load 'config/initializers/resque.rb'

class ResqueCron
  @queue = :cron
end

ARGV.each do |task|
  # Use Time.now instead of Time.current because we don't have Rails.
  # Besides, it gets serialized as a string in redis and Time.zone.parse parses
  # it correctly on the Rails side...
  puts "Enqueuing #{ task } at #{ Time.now }"
  Resque.enqueue(ResqueCron, task, Time.now)
end
```

A neat byproduct of this is that if the cron job fails completely, I can easily queue up the cron jobs with the time they were supposed to run. This works because each part in the system takes a time in as opposed to calling Time.current directly.

### Conclusion

So far this setup has proven extremely reliable for us. Because of the system design, the one time it failed (due to a Ruby version issue) I was able to easily resend all of the emails by re-enqueuing the necessary cron jobs with the relevant time.

The process of delivering emails is something like this:

1. Users get their time zone automatically set just by using the app
2. Cron queues a job in resque for each type of email every hour
3. The resque job compiles a list of users or task reminders that need emails based on the time it was meant to run
4. The resulting worker queues up another resque job for each email that actually needs to be delivered

![Queueing process diagram](/images/timezone_queue.png)

Although it's a fairly complex system for what seems like a simple task, it's important to us that our users can trust they'll get their email when they expect it. This way we can provide a better overall user experience.

If you are a Stride user, and you are not receiving your Monday email between 7am and 8am in your time zone, please let us know.
