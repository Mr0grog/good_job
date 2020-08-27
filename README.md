# GoodJob

GoodJob is a multithreaded, Postgres-based, ActiveJob backend for Ruby on Rails.

Table of Contents
1. Set up
1. GoodJob Configuration
1. Detailed usage and application configuration
    1. Errors and retries
    1. Timeouts
    1. Async
1. Going Deeper
    1. Scaling

**Inspired by [Delayed::Job](https://github.com/collectiveidea/delayed_job) and [Que](https://github.com/que-rb/que), GoodJob is designed for maximum compatibility with Ruby on Rails, ActiveJob, and Postgres to be simple and performant for most workloads.**

- **Designed for ActiveJob.** Complete support for [async, queues, delays, priorities, timeouts, and retries](https://edgeguides.rubyonrails.org/active_job_basics.html) with near-zero configuration. 
- **Built for Rails.** Fully adopts Ruby on Rails [threading and code execution guidelines](https://guides.rubyonrails.org/threading_and_code_execution.html) with [Concurrent::Ruby](https://github.com/ruby-concurrency/concurrent-ruby). 
- **Backed by Postgres.** Relies upon Postgres integrity, session-level Advisory Locks to provide run-once safety and stay within the limits of `schema.rb`, and LISTEN/NOTIFY to reduce queuing latency.
- **For most workloads.** Targets full-stack teams, economy-minded solo developers, and applications that enqueue less than 1-million jobs/day.

For more of the story of GoodJob, read the [introductory blog post](https://island94.org/2020/07/introducing-goodjob-1-0).

<details>
<summary><strong>📊 Comparison of GoodJob with other job queue backends (click to expand)</strong></summary>

|                 | Queues, priority, retries | Database                              | Concurrency       | Reliability/Integrity  | Latency                  |
|-----------------|---------------------------|---------------------------------------|-------------------|------------------------|--------------------------|
| **GoodJob**     | ✅ Yes                     | ✅ Postgres                            | ✅ Multithreaded   | ✅ ACID, Advisory Locks | ✅ Postgres LISTEN/NOTIFY |
| **Que**         | ✅ Yes                     | 🟨 Postgres, requires  `structure.sql` | ✅ Multithreaded   | ✅ ACID, Advisory Locks | ✅ Postgres LISTEN/NOTIFY |
| **Delayed Job** | ✅ Yes                     | ✅ Postgres                            | 🟥 Single-threaded | ✅ ACID, record-based   | 🟨 Polling                |
| **Sidekiq**     | ✅ Yes                     | 🟥 Redis                               | ✅ Multithreaded   | 🟥 Crashes lose jobs    | ✅ Redis BRPOP            |
| **Sidekiq Pro** | ✅ Yes                     | 🟥 Redis                               | ✅ Multithreaded   | ✅ Redis RPOPLPUSH      | ✅ Redis RPOPLPUSH        |

</details>

## Set up

1. Add `good_job` to your application's Gemfile:

    ```ruby
    gem 'good_job'
    ```

1. Install the gem:

    ```bash
    $ bundle install
    ```

1. Run the GoodJob install generator. This will generate a database migration to create a table for GoodJob's job records:
    
    ```bash
    $ bin/rails g good_job:install
    ```

    Run the migration:
    
    ```bash
    $ bin/rails db:migrate
    ```
    
1. Configure the ActiveJob adapter:
    
    ```ruby
    # config/application.rb
    config.active_job.queue_adapter = :good_job
    ```

1. Inside of your application, queue your job 🎉: 
    
    ```ruby
    YourJob.perform_later
    ```
    
    GoodJob supports all ActiveJob features:
    
    ```ruby
    YourJob.set(queue: :some_queue, wait: 5.minutes, priority: 10).perform_later
    ```

1. In development, GoodJob executes jobs immediately. In production, GoodJob provides different options:

    - By default, GoodJob separates job enqueuing from job execution so that jobs can be scaled independently of the web server.  Use the GoodJob command-line tool to execute jobs:
    
        ```bash
        $ bundle exec good_job start
        ```
        
        Ideally the command-line tool should be run on a separate machine or container from the web process. For example, on Heroku:
        
        ```Procfile
        web: rails server
        worker: bundle exec good_job start
        ```
        
        The command-line tool supports a variety of options, see the reference below for command-line configuration.

    - GoodJob can also be configured to execute jobs within the web server process to save on resources. This is useful for low-workloads when economy is paramount.
  
        ```
        $ GOOD_JOB_EXECUTION_MODE=async rails server
        ```
    
        Additional configuration is likely necessary, see the reference below for async configuration.
   
## Configuration

### Command-line options

There several top-level commands available through the `good_job` command-line tool.

Configuration options are available with `help`.

#### `good_job start`

`good_job start` executes queued jobs.

```bash
$ bundle exec good_job help start

Usage:
  good_job start

Options:
  [--max-threads=COUNT]      # Maximum number of threads to use for working jobs. (env var: GOOD_JOB_MAX_THREADS, default: 5)
  [--queues=QUEUE_LIST]      # Queues to work from. (env var: GOOD_JOB_QUEUES, default: *)
  [--poll-interval=SECONDS]  # Interval between polls for available jobs in seconds (env var: GOOD_JOB_POLL_INTERVAL, default: 1)

Executes queued jobs.

All options can be configured with environment variables.
See option descriptions for the matching environment variable name.

== Configuring queues
Separate multiple queues with commas; exclude queues with a leading minus;
separate isolated execution pools with semicolons and threads with colons.
```
    
#### `good_job cleanup_preserved_jobs`

`good_job cleanup_preserved_jobs` deletes preserved job records. See [`GoodJob.preserve_job_records` for when this command is useful.

```
Usage:
  good_job cleanup_preserved_jobs

Options:
  [--before-seconds-ago=SECONDS]  # Delete records finished more than this many seconds ago
                                  # Default: 86400

Deletes preserved job records.

By default, GoodJob deletes job records when the job is performed and this
command is not necessary.

However, when `GoodJob.preserve_job_records = true`, the jobs will be
preserved in the database. This is useful when wanting to analyze or
inspect job performance.

If you are preserving job records this way, use this command regularly
to delete old records and preserve space in your database.
```


### Adapter options

To use GoodJob, you can set `config.active_job.queue_adapter` to a `:good_job` or to an instance of `GoodJob::Adapter`, which you can configure with several options:

- `execution_mode` (symbol) specifies how and where jobs should be executed. You can also set this with the environment variable `GOOD_JOB_EXECUTION_MODE`. It can be any one of:
    - `:inline` executes jobs immediately in whatever process queued them (usually the web server process). This should only be used in test and development environments.
    - `:external` causes the adapter to equeue jobs, but not execute them. When using this option (the default for production environments), you’ll need to use the command-line tool to actually execute your jobs.
    - `:async` causes the adapter to execute you jobs in separate threads in whatever process queued them (usually the web process). This is akin to running the command-line tool’s code inside your web server. It can be more economical for small workloads (you don’t need a separate machine or environment for running your jobs), but if your web server is under heavy load or your jobs require a lot of resources, you should choose `:external` instead.
- `max_threads` (integer) sets the maximum number of threads to use when `execution_mode` is set to `:async`. You can also set this with the environment variable `GOOD_JOB_MAX_THREADS`.
- `queues` (string) determines which queues to execute jobs from when `execution_mode` is set to `:async`. See the description of `good_job start` for more details on the format of this string. You can also set this with the environment variable `GOOD_JOB_QUEUES`.
- `poll_interval` (float) sets the number of seconds between polls for jobs when `execution_mode` is set to `:async`. You can also set this with the environment variable `GOOD_JOB_POLL_INTERVAL`.
 
Using the symbol instead of explicitly configuring the options above (i.e. setting `config.active_job.queue_adapter = :good_job`) is equivalent to:

```ruby
# config/environments/development.rb
config.active_job.queue_adapter = GoodJob::Adapter.new(execution_mode: :inline)

# config/environments/test.rb
config.active_job.queue_adapter = GoodJob::Adapter.new(execution_mode: :inline)

# config/environments/production.rb
config.active_job.queue_adapter = GoodJob::Adapter.new(execution_mode: :external)
```


### Global / code-level options

Good Job’s general behavior can also be configured via several attributes directly on the `GoodJob` module:

- `logger` ([Rails Logger](https://api.rubyonrails.org/classes/ActiveSupport/Logger.html)) lets you set a custom logger for GoodJob. It should be an instance of a Rails `Logger`.
- `preserve_job_records` (boolean) keeps job records in your database even after jobs are completed. (Default: `false`)
- reperform_jobs_on_standard_error (boolean) causes jobs to be re-queued and retried if they raise an instance of `StandardError`. Instances of `Exception`, like SIGINT, will *always* be retried, regardless of this attribute’s value. (Default: `true`)
- `on_thread_error` (proc, lambda, or callable) will be called when a job raises an error. It can be useful for logging errors to bug tracking services, like Sentry or Airbrake.
 
You’ll generally want to configure these in `config/initializers/good_job.rb`, like so:

```ruby
# config/initializers/good_job.rb
GoodJob.on_thread_error = true
GoodJob.reperform_jobs_on_standard_error = false
GoodJob.on_thread_error = -> (exception) { Raven.capture_exception(exception) }
```


## Going deeper


1. Optimize execution to reduce congestion and execution latency. 

    By default, GoodJob creates a single thread execution pool that will execute jobs from any queue. Depending on your application's workload, job types, and service level objectives, you may wish to optimize execution resources; for example, providing dedicated execution resources for transactional emails so they are not delayed by long-running batch jobs. Some options:

    - Multiple execution pools within a single process:
        
        ```bash
        $ bundle exec good_job --queues=transactional_messages:2;batch_processing:1;-transactional_messages,batch_processing:2;* --max-threads=5
        ```
      
        This configuration will result in a single process with 4 isolated thread execution pools. Isolated execution pools are separated with a semicolon (`;`) and queue names and thread counts with a colon (`:`)
        
        - `transactional_messages:2`: execute jobs enqueued on `transactional_messages` with up to 2 threads.
        - `batch_processing:1` execute jobs enqueued on `batch_processing` with a single thread.
        - `-transactional_messages,batch_processing`: execute jobs enqueued on _any_ queue _excluding_ `transactional_messages` or `batch_processing` with up to 2 threads.
        - `*`: execute jobs on any queue on up to 5 threads, as configured by `--max-threads=5`
        
        For moderate workloads, multiple isolated thread execution pools offers a good balance between congestion management and economy. 
        
        Configuration can be injected by environment variables too:
        
        ```bash
        $ GOOD_JOB_QUEUES="transactional_messages:2;batch_processing:1;-transactional_messages,batch_processing:2;*" GOOD_JOB_MAX_THREADS=5 bundle exec good_job
        ```
        
    - Multiple processes; for example, on Heroku:
    
        ```procfile
        # Procfile
        
        # Separate dyno types
        worker: bundle exec good_job --max-threads=5
        transactional_worker: bundle exec good_job --queues=transactional_messages --max-threads=2
        batch_worker: bundle exec good_job --queues=batch_processing --max-threads=1
      
        # Combined multi-process dyno
        combined_worker: bundle exec good_job --max-threads=5 & bundle exec good_job --queues=transactional_messages --max-threads=2 & bundle exec good_job --queues=batch_processing --max-threads=1 & wait -n
        ```
      
      Running multiple processes can optimize for CPU performance at the expense of greater memory and system resource usage.

    _Keep in mind, queue operations and management is an advanced discipline. This stuff is complex, especially for heavy workloads and unique processing requirements. Good job 👍_




### Error handling, retries, and reliability

GoodJob guarantees that a completely-performed job will run once and only once. GoodJob fully supports ActiveJob's built-in functionality for error handling, retries and timeouts. Writing reliable, transactional, and idempotent `ActiveJob#perform` methods is outside the scope of GoodJob.

#### Error handling

By default, if a job raises an error while it is being performed, _and it bubbles up to the GoodJob backend_, GoodJob will be immediately re-perform the job until it finishes successfully.

- `Exception`-type errors, such as a SIGINT, will always cause a job to be re-performed.
- `StandardError`-type errors, by default, will cause a job to be re-performed, though this is configurable:
   
    ```ruby
    # config/initializers/good_job.rb
    GoodJob.reperform_jobs_on_standard_error = true # => default
    ```

To report errors that _do_ bubble up to the GoodJob backend, assign a callable to `GoodJob.on_thread_error`. For example:

```ruby
# config/initializers/good_job.rb

# With Sentry (or Bugsnag, Airbrake, Honeybadger, etc.)
GoodJob.on_thread_error = -> (exception) { Raven.capture_exception(exception) }
```

### Retrying jobs

ActiveJob can be configured to retry an infinite number of times, with an exponential backoff. Using ActiveJob's `retry_on` will ensure that errors do not bubble up to the GoodJob backend:

```ruby
class ApplicationJob < ActiveJob::Base  
  retry_on StandardError, wait: :exponentially_longer, attempts: Float::INFINITY
  # ...
end
```

When specifying a limited number of retries, care must be taken to ensure that an error does not bubble up to the GoodJob backend because that will result in the job being re-performed:

```ruby
class ApplicationJob < ActiveJob::Base  
  retry_on StandardError, attempts: 5 do |_job, _exception|
    # Log error, etc.
    # You must implement this block, otherwise, 
    #   Active Job will re-raise the error.
    # Do not re-raise the error, otherwise 
    #   GoodJob will immediately re-perform the job. 
  end
  # ...
end
```

GoodJob can be configured to allow omitting `retry_on`'s block argument and implicitly discard un-handled errors:

```ruby
# config/initializers/good_job.rb

# Do NOT re-perform a job if a StandardError bubbles up to the GoodJob backend
GoodJob.reperform_jobs_on_standard_error = false 
```

When using an exception monitoring service (e.g. Sentry, Bugsnag, Airbrake, Honeybadger, etc), the use of `rescue_on` may be incompatible with their ActiveJob integration. It's safest to explicitly wrap jobs with an exception reporter. For example:

```ruby
class ApplicationJob < ActiveJob::Base  
  retry_on StandardError, wait: :exponentially_longer, attempts: Float::INFINITY
  
  around_perform do |_job, block|
    block.call
  rescue StandardError => e
    Raven.capture_exception(e)
    raise
  end
  # ...
end
```

ActiveJob's `discard_on` functionality is supported too.

#### ActionMailer retries

Using a Mailer's `#deliver_later` will enqueue an instance of `ActionMailer::DeliveryJob` which inherits from `ActiveJob::Base` rather than your applications `ApplicationJob`. You can use an initializer to configure retries on `ActionMailer::DeliveryJob`:

```ruby
# config/initializers/good_job.rb
ActionMailer::DeliveryJob.retry_on StandardError, wait: :exponentially_longer, attempts: Float::INFINITY

# With Sentry (or Bugsnag, Airbrake, Honeybadger, etc.)
ActionMailer::DeliveryJob.around_perform do |_job, block|
  block.call
rescue StandardError => e
  Raven.capture_exception(e)
  raise
end
```

#### Timeouts

Job timeouts can be configured with an `around_perform`:

```ruby
class ApplicationJob < ActiveJob::Base  
  JobTimeoutError = Class.new(StandardError)
  
  around_perform do |_job, block|
    # Timeout jobs after 10 minutes
    Timeout.timeout(10.minutes, JobTimeoutError) do
      block.call
    end
  end
end
```

### Configuring job execution threads
    
GoodJob executes enqueued jobs using threads. There is a lot than can be said about [multithreaded behavior in Ruby on Rails](https://guides.rubyonrails.org/threading_and_code_execution.html), but briefly:

- Each GoodJob execution thread requires its own database connection, which are automatically checked out from Rails’s connection pool. _Allowing GoodJob to schedule more threads than are available in the database connection pool can lead to timeouts and is not recommended._ 
- The maximum number of GoodJob threads can be configured, in decreasing precedence:
    1. `$ bundle exec good_job --max_threads 4`
    2. `$ GOOD_JOB_MAX_THREADS=4 bundle exec good_job`
    3. `$ RAILS_MAX_THREADS=4 bundle exec good_job`
    4. Implicitly via Rails's database connection pool size (`ActiveRecord::Base.connection_pool.size`)

### Executing jobs async / in-process

GoodJob is able to run "async" in the same process as the webserver (e.g. `bin/rail s`). GoodJob's async execution mode offers benefits of economy by not requiring a separate job worker process, but with the tradeoff of increased complexity. Async mode can be configured in two ways:

- Directly configure the ActiveJob adapter:

    ```ruby
    # config/environments/production.rb
    config.active_job.queue_adapter = GoodJob::Adapter.new(execution_mode: :async, max_threads: 4, poll_interval: 30)
    ```
- Or, when using `...queue_adapter = :good_job`, via environment variables:

    ```bash
    $ GOOD_JOB_EXECUTION_MODE=async GOOD_JOB_MAX_THREADS=4 GOOD_JOB_POLL_INTERVAL=30 bin/rails server
    ```
 
Depending on your application configuration, you may need to take additional steps:

- Ensure that you have enough database connections for both web and job execution threads:

    ```yaml
    # config/database.yml
    pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5).to_i + ENV.fetch("GOOD_JOB_MAX_THREADS", 4).to_i %>
    ```

- When running Puma with workers (`WEB_CONCURRENCY > 0`) or another process-forking webserver, GoodJob's threadpool schedulers should be stopped before forking, restarted after fork, and cleanly shut down on exit. Stopping GoodJob's scheduler pre-fork is recommended to ensure that GoodJob does not continue executing jobs in the parent/controller process. For example, with Puma:

    ```ruby
    # config/puma.rb
  
    before_fork do
      GoodJob.shutdown
    end
    
    on_worker_boot do
      GoodJob.restart
    end
    
    on_worker_shutdown do
      GoodJob.shutdown
    end
  
    MAIN_PID = Process.pid
    at_exit do
      GoodJob.shutdown if Process.pid == MAIN_PID
    end
    ```
  
  GoodJob is compatible with Puma's `preload_app!` method.
  
### Migrating to GoodJob from a different ActiveJob backend

If your application is already using an ActiveJob backend, you will need to install GoodJob to enqueue and perform newly created jobs _and_ finish performing pre-existing jobs on the previous backend.

1. Enqueue newly created jobs on GoodJob either entirely by setting `ActiveJob::Base.queue_adapter = :good_job` or progressively via individual job classes:

    ```ruby
    # jobs/specific_job.rb
    class SpecificJob < ApplicationJob
      self.queue_adapter = :good_job
      # ...
    end
    ```

1. Continue running executors for both backends. For example, on Heroku it's possible to run [two processes](https://help.heroku.com/CTFS2TJK/how-do-i-run-multiple-processes-on-a-dyno) within the same dyno:
    
   ```procfile
    # Procfile
    # ...
    worker: bundle exec que ./config/environment.rb & bundle exec good_job & wait -n
    ```

1. Once you are confident that no unperformed jobs remain in the previous ActiveJob backend, code and configuration for that backend can be completely removed.

### Monitoring and preserving worked jobs

GoodJob is fully instrumented with [`ActiveSupport::Notifications`](https://edgeguides.rubyonrails.org/active_support_instrumentation.html#introduction-to-instrumentation).

By default, GoodJob will delete job records after they are run, regardless of whether they succeed or not (raising a kind of `StandardError`), unless they are interrupted (raising a kind of `Exception`). 

To preserve job records for later inspection, set an initializer:

```ruby
# config/initializers/good_job.rb
GoodJob.preserve_job_records = true
```

It is also necessary to delete these preserved jobs from the database after a certain time period:

- For example, in a Rake task:
  
    ```ruby
    GoodJob::Job.finished(1.day.ago).delete_all
    ```

- For example, using the `good_job` command-line utility:

    ```bash
    $ bundle exec good_job cleanup_preserved_jobs --before-seconds-ago=86400
    ```

<!-- Understanding THE ADAPTER -->
    
    By default, using `:good_job` is equivalent to manually configuring the adapter:
    
    ```ruby
    # config/environments/development.rb
    config.active_job.queue_adapter = GoodJob::Adapter.new(execution_mode: :inline)
   
    # config/environments/test.rb
    config.active_job.queue_adapter = GoodJob::Adapter.new(execution_mode: :inline)
   
    # config/environments/production.rb
    config.active_job.queue_adapter = GoodJob::Adapter.new(execution_mode: :external)
    ```

## Contributing

Contributions are welcomed and appreciated 🙏

- Review the [Prioritized Project Backlog](https://github.com/bensheldon/good_job/projects/1).
- Open a new Issue or contribute to an [existing Issue](https://github.com/bensheldon/good_job/issues). Questions or suggestions are fantastic.
- Participate according to our [Code of Conduct](https://github.com/bensheldon/good_job/projects/1).

### Gem development

To run tests:

```bash
# Clone the repository locally
$ git clone git@github.com:bensheldon/good_job.git

# Set up the local environment
$ bin/setup

# Run the tests
$ bin/rspec
```

This gem uses Appraisal to run tests against multiple versions of Rails:

```bash
# Install Appraisal(s) gemfiles
$ bundle exec appraisal

# Run tests
$ bundle exec appraisal bin/rspec
```

For developing locally within another Ruby on Rails project:

```bash
# Within Ruby on Rails directory...
$ bundle config local.good_job /path/to/local/git/repository

# Confirm that the local copy is used
$ bundle install

# => Using good_job 0.1.0 from https://github.com/bensheldon/good_job.git (at /Users/You/Projects/good_job@dc57fb0)
```

### Releasing

Package maintainers can release this gem by running:

```bash
# Sign into rubygems
$ gem signin

# Add a .env file with the following:
# CHANGELOG_GITHUB_TOKEN= # Github Personal Access Token

# Update version number, changelog, and create git commit:
$ bundle exec rake release[minor] # major,minor,patch

# ..and follow subsequent directions. 
```

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
