Delayed::Job
============

Delayed_job (or DJ) encapsulates the common pattern of asynchronously executing longer tasks in the background.

It is a direct extraction from Shopify where the job table is responsible for a multitude of core tasks. Amongst those tasks are:

* sending massive newsletters
* image resizing
* http downloads
* updating smart collections
* updating solr, our search server, after product changes
* batch imports
* spam checks

What is this fork for?
----------------------

My purpose with this fork is make delayed_job (1) for flexible, customize how your workers behave, and (2) use threads to launch the jobs.

The common use will be to have several worker running concurrently (in one process) and each one with differents constraints so they'll run different kind of jobs. Or to have one worker launching several threads, grouping the jobs for one attribute of them so only one of each are in execution.


Setup
-----

The library evolves around a delayed_jobs table which looks as follows (do not forget to add you own fields according to your needs):

    create_table :delayed_jobs, :force => true do |table|
        table.integer  :priority, :default => 0      # Allows some jobs to jump to the front of the queue
        table.integer  :attempts, :default => 0      # Provides for retries, but still fail eventually.
        table.text     :handler                      # YAML-encoded string of the object that will do work
        table.string   :job_type                     # Class name of the job object, for type-specific workers
        table.string   :name                         # The display name, an informative field or can be use to filter jobs
        table.string   :last_error                   # reason for last failure (See Note below)
        table.datetime :run_at                       # When to run. Could be Time.now for immediately, or sometime in the future.
        table.datetime :locked_at                    # Set when a client is working on this object
        table.datetime :failed_at                    # Set when all retries have failed (actually, by default, the record is deleted instead)
        table.string   :locked_by                    # Who is working on this object (if locked)
        table.datetime :finished_at			 # Used for statiscics / monitoring
        table.timestamps
    end

You can generate the migration executing:

    $ script/generate delayed_job
      exists  db/migrate
      create  db/migrate/20090807090217_create_delayed_jobs.rb


On failure, the job is scheduled again in 5 seconds + N ** 4, where N is the number of retries.

The default `MAX_ATTEMPTS` is 25 (jobs can override this value by responding to `:max_attempts`). After this, the job either deleted (default), or left in the database with "failed_at" set. With the default of 25 attempts, the last retry will be 20 days later, with the last interval being almost 100 hours.

The default `MAX_RUN_TIME` is 4.hours. If your job takes longer than that, another computer could pick it up. It's up to you to make sure your job doesn't exceed this time. You should set this to the longest time you think the job could take.

By default, it will delete failed jobs. If you want to keep failed jobs, set `Delayed::Job.destroy_failed_jobs = false`. The failed jobs will be marked with non-null failed_at.

Same thing for successful jobs. They're deleted by default and, to keep them, set `Delayed::Job.destroy_successful_jobs = false`. They will be marked with finished_at. This is useful for gathering statistics like how long jobs took between entering the queue (created_at) and being finished (finished_at).

You have several named scopes for searching unfinished/finsihed/failed jobs, very useful when destroy_successful_jobs is false `Delayed::Job.unfinished`, `Delayed::Job.finsihed`, `Delayed::Job.failed`.

Here is an example of changing job parameters in Rails:

    # config/initializers/delayed_job_config.rb
    Delayed::Job.destroy_failed_jobs     = false
    Delayed::Job.destroy_successful_jobs = false
    silence_warnings do
      Delayed::Job.const_set("MAX_ATTEMPTS", 3)
      Delayed::Job.const_set("MAX_RUN_TIME", 5.hours)

      Delayed::Worker.const_set("SLEEP", 5.minutes.to_i)
    end

Note: If your error messages are long, consider changing last_error field to a :text instead of a :string (255 character limit).


Usage
-----

Jobs are simple ruby objects with a method called perform. Any object which responds to perform can be stuffed into the jobs table.
Job objects are serialized to yaml so that they can later be resurrected by the job runner.

    class NewsletterJob < Struct.new(:text, :emails)
      def perform
        emails.each { |e| NewsletterMailer.deliver_text_to_email(text, e) }
      end
    end

    Delayed::Job.enqueue NewsletterJob.new('lorem ipsum...', Customers.find(:all).collect(&:email))

There is also a second way to get jobs in the queue: send_later.

    BatchImporter.new(Shop.find(1)).send_later(:import_massive_csv, massive_csv)

And also you can specified priority as second parameter and the time the job should execute as thrird one


    class FooJob
      def perform
        ...
      end
    end

    important_job = FooJob.new
    normal_job    = FooJob.new

    # Delayed::Job.enqueue( job, priority, start_at )
    Delayed::Job.enqueue important_job, 100
    Delayed::Job.enqueue normal_job, 1, 2.hours.from_now

This will simply create a `Delayed::PerformableMethod` job in the jobs table which serializes all the parameters you pass to it. There are some special smarts for active record objects which are stored as their text representation and loaded from the database fresh when the job is actually run later.


Running the jobs
----------------

You can invoke `rake jobs:work` which will start working off jobs. You can cancel the rake task with `CTRL-C`.

You can also run by writing a simple `script/job_runner`, and invoking it externally:


    require File.dirname(__FILE__) + '/../config/environment'

    Delayed::Worker.new.start

Workers can be running on any computer, as long as they have access to the database and their clock is in sync. You can even run multiple workers on per computer, but you must give each one a unique name.


    require File.dirname(__FILE__) + '/../config/environment'
    N = 10
    workers = []
    N.times do |n|
    workers << Thread.new do
      Delayed::Worker.new( :name => "Worker #{n}" ).start
    end
    end

    workers.first.join # wait until finish (signal catched)

Keep in mind that each worker will check the database at least every 5 seconds.

Note: The rake task will exit if the database has any network connectivity problems.

If you only want to run specific types of jobs in a given worker, include them when initializing the worker:

    Delayed::Worker.new(:job_types => "SimpleJob").start
    Delayed::Worker.new(:job_types => ["SimpleJob", "NewsletterJob"]).start

Also for a more specific restriction you can define in your job's classes a `display_name` method, and create workers to specific kind of jobs

    # 1 - The job class that does the real work
    class MyJob
      def initialize( data )
        @some_data = data
      end

      def perform
        # do the real work
      end

      def display_name
        "foobar #{@some_data}"
      end
    end

    # 2 - Enqueue jobs
    Delayed::Job.enqueue MyJob.new("foobar")
    Delayed::Job.enqueue MyJob.new("arrrr")

    # 3 - Create workers, one for each type of "data"
    Thread.new {
      # This worker will only perform jobs which display name is like "%foobar%"
      Delayed::Worker.new :name => "Worker for foobar", :only_for => "foobar"
    }
    Thread.new {
      Delayed::Worker.new :name => "Worker for arrr", :only_for => "arrr"
    }


Group by property
-----------------
In 2.1.0 version there is a new way to operate really different from the previous model. Instead of having one (or several) worker with some restrictions, we'll have only ONE worker launching jobs in threads. The jobs will be classified by the :group_by property, and only one of each group will be in execution at a time. Let's see an example:

    $ script/generate model device
    $ script/generate migration add_device_id_to_delayed_job_table # do migration, adding t.references :device

    Delayed::Worker.new :group_by => :device

Our jobs will have a device method (because we have done the relationship adding the foreign_key to the table of jobs). If we have 2 jobs for the same "device" and n more jobs, each one for different devices, at first the latter ones will be executed plus one of the two jobs of the same device. After that, the other job will be executed. One at a time.

Cleaning up
-----------

You can invoke `rake jobs:clear` to delete all jobs in the queue.

Changes
-------
* 2.1.0: group_by property so we have one worker with n threads, one per "group".
* 2.0.7: Save the last_error field for non-repeatable jobs (the ones with self.attempts == 0)
* 2.0.6: You can pass to worker any kind of option, and you can override conditions_avilable method to customize the behaviour and the searches of jobs to execute
* 2.0.4: Add `Delayed::HIDE_BACKTRACE` for showing only the first line of the errors avoiding long backtraces
* 2.0.3: The only_for feature wasn't working at all
* 2.0.2: Only update run_at when it's gonna be executed another time for sure (attempts < max_attempts)
* 2.0.1: named_scope Delayed::Job.failed/finished/unfinished (jobs that have failed, have finished ok or still haven't been done)
* 2.0.0: Contains the changes made in this fork, the ability to create workers with individual constraints without interfere to other workers

* 1.7.0: Added failed_at column which can optionally be set after a certain amount of failed job attempts. By default failed job attempts are destroyed after about a month.

* 1.6.0: Renamed locked_until to locked_at. We now store when we start a given job instead of how long it will be locked by the worker. This allows us to get a reading on how long a job took to execute.

* 1.5.0: Job runners can now be run in parallel. Two new database columns are needed: locked_until and locked_by. This allows us to use   pessimistic locking instead of relying on row level locks. This enables us to run as many worker processes as we need to speed up queue processing.

* 1.2.0: Added #send_later to Object for simpler job creation

* 1.0.0: Initial release
