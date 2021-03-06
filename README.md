# Rails 5 Deadlock Example

I am seeing a deadlocking issue that appears to have something to do
with threads and autoloading.

This example app uses the Parallel gem, https://github.com/grosser/parallel,
to run three threads from my `ParallelMapper` class. Each thread creates an instance of my `CollaboratorOne` class to perform work. In turn `CollaboratorOne` creates an instance of `CollaboratorTwo` to assist with the work.

```
class ParallelMapper
  def work_in_threads
    data = [1, 2, 3]
    Parallel.map(data, in_threads: 3) do |item|
      ActiveRecord::Base.connection_pool.with_connection do |conn|
        puts "ParallelMapper instance: work_in_threads: #{item}"
        collaborator_one = CollaboratorOne.new(item)
        collaborator_one.perform
      end
    end
  end
end
```

I have `config.eager_load = false` in config/environments/test.rb|development.rb.
Therefore the instantiations of `CollaboratorOne` and `CollaboratorTwo` must provoke autoloading of those classes unless I explicitly require them from `ParallelMapper`.

Finally I have a request spec that makes an HTTP request to my "/examples" endpoint.
The `ExamplesController` creates a `ParallelMapper` instance and calls its `work_in_threads` method to kick off the work described above.

```
require "rails_helper"

describe "request where the controller creates threads", threads: true do
  it "does what we expect" do
    get examples_url
    expect(response.code).to eq("200")
  end
end
```

* The Rspec test throws an error "No live threads left. Deadlock?"
* Manually loading http://localhost:3000/examples in my browser actually deadlocks - the server hangs while processing the request.

Why does this happen and how should I address it?

If I explicitly require the collaborator classes the problem goes away.
That seems a little haphazard, but is it the correct solution?

Setting `config.eager_load = true` in test/development would also avoid the problem although I'd prefer not to do that for speed.

The `Rails.application.reloader.wrap` API introduced by @matthewd in https://github.com/rails/rails/pull/23807
might be helpful although I'm not sure how to apply it to my problem here.
Maybe the parallel gem would need to make use of that API rather than my app?

### Request spec that throws thread error ###
```
$ bundle exec rspec spec/requests/examples_spec.rb
ParallelMapper instance: work_in_threads: 2
ParallelMapper instance: work_in_threads: 1
ParallelMapper instance: work_in_threads: 3
CollaboratorOne instance: perform 2CollaboratorOne instance: perform 3

F
CollaboratorTwo instance: assist 3CollaboratorOne instance: perform 1
CollaboratorTwo instance: assist 1


Failures:

  1) request where the controller creates threads does what we expect
     Failure/Error:
       Parallel.map(data, in_threads: 3) do |item|
         ActiveRecord::Base.connection_pool.with_connection do |conn|
           puts "ParallelMapper instance: work_in_threads: #{item}"
           collaborator_one = CollaboratorOne.new(item)
           collaborator_one.perform
         end
       end

     fatal:
       No live threads left. Deadlock?
     # ./app/models/parallel_mapper.rb:9:in `work_in_threads'
     # ./app/controllers/examples_controller.rb:4:in `index'
     # ./spec/requests/examples_spec.rb:6:in `block (2 levels) in <top (required)>'

Finished in 0.549 seconds (files took 6.04 seconds to load)
1 example, 1 failure

Failed examples:

rspec ./spec/requests/examples_spec.rb:4 # request where the controller creates threads does what we expect
```

### Manual test that deadlocks ###
```
$ bundle exec rails server
=> Booting Puma
=> Rails 5.0.0.1 application starting in development on http://localhost:3000
=> Run `rails server -h` for more startup options
Puma starting in single mode...
* Version 3.6.0 (ruby 2.3.1-p112), codename: Sleepy Sunday Serenity
* Min threads: 5, max threads: 5
* Environment: development
* Listening on tcp://localhost:3000
Use Ctrl-C to stop

# Visit http://localhost:3000/examples in browser...

Started GET "/examples" for ::1 at 2016-11-11 13:20:42 -0500
Processing by ExamplesController#index as HTML
ParallelMapper instance: work_in_threads: 3
ParallelMapper instance: work_in_threads: 1
ParallelMapper instance: work_in_threads: 2

# ... deadlocked ...
```

### Environment ###
```
$ ruby -v
ruby 2.3.1p112 (2016-04-26 revision 54768) [x86_64-darwin15]

$ rails -v
Rails 5.0.0.1
$ rails new rails-deadlock-example -d postgresql

$ uname -prsv
Darwin 15.6.0 Darwin Kernel Version 15.6.0: Thu Sep  1 15:01:16 PDT 2016; root:xnu-3248.60.11~2/RELEASE_X86_64 i386
```

### How this app was created ###
```
$ rails new rails-deadlock-example -d postgresql
$ rails generate rspec:install
```

### DB Setup ###
```
$ psql -d postgres
psql (9.6.0)
Type "help" for help.

postgres=# create database rails_deadlock_example_development;
CREATE DATABASE
postgres=# \q

$ rails db:setup
```

