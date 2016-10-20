# Ruby Performance Optimization Book Abstract

## Try to disable GC and define memory consumption

1. GC often makes Ruby slow (especialy <= v2.0 ). And that's because of high memory consumption
2. Ruby has significant memory overhead
3. GC in v2.1+ is 5 times faster!
4. Raw performance of v1.9 - v2.3 is about the same

See [001_gc.rb](001_gc.rb)

## Try to optimize memory

1. 80% of performance optimization comes from memory optimization
  See [002_memory.rb](002_memory.rb)

2. GC::Profiler has memory and CPU overhead (See [wrapper.rb](wrapper.rb) for custom wrapper example).

3. Save memory by avoiding copiyng objects, modify them in place if it is possible (use ! methods).

  See [004_string_bang.rb](004_string_bang.rb)

4. If your String is less than 40 bytes - user `<<`, not `+=` method to concatenate it and Ruby will not allocate additional object.

  See [004_array_bang.rb](004_array_bang.rb) (w/ GC)

5. Read files line by line. And keep in mind not only total memory consumption but also peaks.

  See [005_files.rb](005_files.rb) (w/ and w/o GC)

6. Callbacks cause object to stay in memory. If you store callbacks, do not forget to remove them after they called.
  See [006_callbacks_1.rb](006_callbacks_1.rb)
  See [006_callbacks_2.rb](006_callbacks_2.rb)
  See [006_callbacks_3.rb](006_callbacks_3.rb)
  Try to avoid `&block` and use `yield` instead.

7. Iterators use block arguments, so use them carefuly

  Issues:
  1. GC will not collect iterable before iterator is finished
  2. Iterators create temp objects

  Solutions:

  1. Free objects from collection during iteration & use `each!` pattern
    See [007_iter_1.rb](007_iter_1.rb)
  2. Look at C code to find object allocations
    See [007_iter_2.rb](007_iter_2.rb) (for ruby < 2.3.0)

    Table of `T_NODE` allocations per iterator item for ruby 2.1:

    | Iterator         | Enum | Array | Range |  
    | ---------------: | ---- | ----- | ----- |  
    | all?             | 3    | 3     | 3     |  
    | any?             | 2    | 2     | 2     |  
    | collect          | 0    | 1     | 1     |  
    | cycle            | 0    | 1     | 1     |  
    | delete_if        | 0    | —     | 0     |  
    | detect           | 2    | 2     | 2     |  
    | each             | 0    | 0     | 0     |  
    | each_index       | 0    | —     | —     |  
    | each_key         | —    | —     | 0     |  
    | each_pair        | —    | —     | 0     |  
    | each_value       | —    | —     | 0     |  
    | each_with_index  | 2    | 2     | 2     |  
    | each_with_object | 1    | 1     | 1     |  
    | fill             | 0    | —     | —     |  
    | find             | 2    | 2     | 2     |  
    | find_all         | 1    | 1     | 1     |  
    | grep             | 2    | 2     | 2     |  
    | inject           | 2    | 2     | 2     |  
    | map              | 0    | 1     | 1     |  
    | none?            | 2    | 2     | 2     |  
    | one?             | 2    | 2     | 2     |  
    | reduce           | 2    | 2     | 2     |  
    | reject           | 0    | 1     | 0     |  
    | reverse          | 0    | —     | —     |  
    | reverse_each     | 0    | 1     | 1     |  
    | select           | 0    | 1     | 0     |  

8. Date parsing is slow
  See [008_date_parsing.rb](008_date_parsing.rb)

9. `Object#class`, `Object#is_a?`, `Object#kind_of?` are slow when used inside iterators

10. Use SQL for aggregation & calculation if it is possible
  See [009_db](009_db) (database query itself is only 30ms)

11. Use native (compiled C) gems if possible

## Optimize Rails

1. ActiveRecord uses 3x DB data size memory and calls often calls GC

2. Use `#pluck`, `#select` to load only necessary data

3. Preload associations if you plan to use them

4. Use `#find_by_sql` to aggregate associations data

5. Use `#find_each` & `#find_in_batches`

6. Use `ActiveRecord::Base.connection.execute`, `ActiveRecord::Base.connection.exec_query`, `ActiveRecord::Base.connection.select_values`, `#update_all` to perform simple operations

7. Use `render partial: 'a', collection: @col`, which loads partial template only once

8. Paginate large views

9. You may disable logging to increase performance

10. Watch your helpers, they may be iterators-unsafe

## Profiling

_Profiling = measuring CPU/Memory usage + interpreting results_

__For CPU profiling disable GC!__

### ruby-prof

__ruby-prof__ gem has both API (for isolated profiling) and CLI (for startup profiling). It also has Rack Middleware for Rails.

See [010_rp_1.rb](010_rp_1.rb)

Some programs may spend more time on startup than on actual code execution.
Sometimes `GC.disable` may take significant amount of time because of _lazy GC sweep_.

Use `Rack::RubyProf` middleware to profile Rails apps. Include it before `Rack::Runtime` to include other middlewares in report.
To disable GC, use custom middleware [010_rp_rails/config/application.rb](010_rp_rails/config/application.rb).

Rails profiling best practices:

1. Disable GC
2. Always profile in _production_ mode
3. Profile twice and discard cold-start results
4. Profile w/ & w/o caching if you use it

The most useful report types for ruby-prof (see [011_rp_rep.rb](011_rp_rep.rb)):

1. __Flat__ (Shows which functions are slow)
2. __Call graph__ (Shows callers and callees)
3. __Stack report__ (Shows execution paths; good for small chunks of code)

#### Callgrind format

Ruby-prof can generate callgrind files with CallTreePrinter (see [011_rp_rep.rb](011_rp_rep.rb)).  
Callgrind profiles have double counting ~~issue~~!.  
Callgrind profiles show loops as recursion.  
It is better to start from the bottom of Call Graph and optimize its leaves first.

## Optimizing with Profiler

Always start optimizing with writing tests & benchmarks.  
**!** Profiler adds up to **10x** time to function calls.  
If you optimized individual functions but the whole thing is still slow, look at the code at a higher abstraction level.

Optimization tips:

1. Optimization with the profiler is a craft (not engineering)
2. Always write test
3. Never forget about the big picture
4. Profiler obscures measurements, benchmarks needed

## Profile Memory

80% of ruby performance optimization come from memory optimization.

You have 3 options for memory profiling:

1. Massif / Stackprof profiles
2. Patched Ruby interpreter & ruby-prof
3. Printing `GC#stat` & `GC::Profiler` measurements

### Specific tools

To detect if memory profiling needed you should use *monitoring* and *profiling* tools.  
Good tool for profiling is **Valgrind Massif** but it shows memory allocations only for C/C++ code.

Another tool is **Stackprof** that shows shows number of object allocations (that is proportional to memory consumption) (see [014_stackprof.rb](014_stackprof.rb)). But if your code allocates small number of large objects, it won't help.  
Stackprof could generate flamegraphs and it's OK to use it in production, because it has no overhead.

### Patched Ruby & RubyProf

You need RailsExpress patched Ruby (google it). Then set RubyProf *measure mode* and use one of printers (see [015_rp_memory.rb](015_rp_memory.rb)). Don't forget to enable memory stats with `GC.enable_stats`.

Modes for memory profiling:

* MEMORY - mem usage
* ALLOCATIONS - # of object allocations
* GC_RUNS - # of GC runs (useless for optimization)
* GC_TIME - GC time (useless for optimization)

Memory profile shows only new memory allocations (not total number at time) and doesn't show GC reclaims.

**!** Ruby allocates temp object for string > 23 chars.

### Manual way

We can measure current memory usage, but it is not very useful.

On Linux we can use OS tools:

~~~ruby
memory_before = `ps -o rss= -p #{Process.pid}`.to_i / 1024
do_something
memory_after = `ps -o rss= -p #{Process.pid}`.to_i / 1024
~~~

`GC#stat` and `GC::Profiler` can reveal some information.

## Measure

For adequate measurements we should measure a number of times and take median value.  
A lot of external (CPU, OS, latency, etc.) and internal (GC runs, etc.) factors affect measured numbers. It is impossible to entirely exclude them.

### Minimize external factors

* Disable dynamic CPU frequency (governor, cpupower in Linux)
* Warm up machine

### Minimize internal factors

Two things can affect application: GC and System calls (including I/O calls).

You may disable GC for measurements or force it before benchmark with `GC.start` (but not in a loop **!** because of new object being created in it).  
On Linux & Mac OS process fork available to fix that issue:

~~~ruby
100.times do
  GC.start
  pid = fork do
    GC.start
    m = Benchmark.realtime { ... }
  end

  Process.waitpid(pid)
end
~~~

### Analyze with Statistics

*Confidence interval* - interval within which we can confidently state the true optimization lies.  
*Level of confidence* - the size of confidence interval.

Analysis algorithm:

1. Estimate means:

  `mx = sum(xs) / count(xs); my = sum(ys) / count(ys)`

2. Calculate standard deviation:

  `sdx = sqrt(sum(sqr(xi - mx)) / count(xs) - 1); sdy = sqrt(sum(sqr(yi - my)) / count(ys) - 1)` 

3. Calculate optimization mean:

  `mo = mx - my`

4. Calculate standard error:

  `err = sqrt(sqr(sdx) / count(xs) + sqr(sdy) / count(ys))`

5. Get the confidence interval:

  `interval = (mo - 2 * err)..(mo + 2 * err)`

Both lower and upper bounds of confidence interval should be > 0 (see [016_statistics.rb](016_statistics.rb)). Always make more than **30** measurements for good results.

For Ruby, round measures to tenth of milliseconds (e.g. 1.23 s). For rounding use tie-breaking "round half to even" rule (round 5 to even number: 1.25 ~= 1.2, 1.35 ~= 1.4).

For better results you may exclude outliers and first (cold) measure results.

### Test Rails performance

For Rails performance testing use special gems (e.g. *rails-perftest*) or write your own custom assertions.

Tips:

* It is a good practice to create Rails performance *integration* tests
* But don't forget to turn on caching and set log level to `:info`
* Database size may affect your results, so rollback transactions or clear data
* Write performance test for complex DB queries
* Test DB queries count and try to reduce it (may use *assert_value* gem) (see [017_query_count.rb](017_query_count.rb))
* Generate enough data for performance test

