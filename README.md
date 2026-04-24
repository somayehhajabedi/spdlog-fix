
A tiny project that uses spdlog.
Build:
  cmake -S . -B build
  cmake --build build -j
  ./build/spdlogger --seconds 5 (optional --stress)


Your task 
Identify and fix as many correctness issues as you reasonably can. 
For each fix, briefly explain: 
what the problem was 
why it happens (especially why it’s intermittent) 
why your fix is correct 

Constraints 
Do not remove async logging. 

Upload solutions to github with Readme 


/*************** Issues and fixes ***************/


1. level.h  : Data race
   
   The problem in level.h is "g_level" variable has been read by first thread and changed by second thread and they did not use synchronization between them.
   It is not correct because this is a data race leads to undefined behavior, but It is intermittent because since the behavior depends on thread scheduling 
   and timing, so it may be correct and sometimes fails. 
   For fixing: changed g_level from int to std::atomic<int> and used atomic operations.
   This fix is correct because atomic operations remove the race and make concurrent access safe.

2. shutdown.h  :  Logger lifetime / shutdown race

   The problem in shutdown.h is . The pointer to the logger is used by a worker thread while spdlog::shutdown() may run concurrently. 
   It is not correct because the thread is still using the logger while shutdown destroys it, this causes a use-after-free. 
   It is intermittent because the failure depends on timing between the logging thread and shutdown. Some runs may appear fine, while others may hit use-after-free behavior.  
   For fixing: Added shutdown only after all threads are stopped and joined.
   Why the fix is correct: this guarantees proper resource lifetime and prevents concurrent use of a logger during destruction.

   stop = true
   join all threads
   spdlog::shutdown()

4. sink_callback.h :   recursive / stack overflow / double logging / crash

   The problem is the custom sink calls spdlog::info() function from inside sink_it_() function.
   That can lead to recursion, crashs or deadlock based on timing.
   It is intermittent because failure depends on timing, and internal locking, it may show up as recursive logging, stack overflow or a crash.
   For fixing: Removed logging calls from inside sink_it_() function.
   This fix is correct because removes recursive entry into the logging path and prevents recursion and crash.

6. string_uaf.h  Use after free and dangling pointer
   Problem: The code saves tmp.data() in a pointer ptr but tmp is local and will be destroyed after function set() finish executing.
   This is problem because tmp's lifetime is shorter than the later use in emit() function.
   Why it is intermittent: the pointer may still work if it has not yet been reused, but in later runs may produce damaged output or crash based on timing and memory reuse.
   For fixing: replaced the pointer with a std::string stored inside DeferredLog struct, in set() we construct msg and in emit() we use it.
   This fix is correct because the message is valid for the lifetime of the DeferredLog object, so emit() function always uses valid data.

7. timebase.h  : Using steady_clock and system_clock together
   The code uses steady_clock and system_clock to calculate duration, this is incorrect because they are not comparable. This cause wrong values.
   Fix: used steady_clock for both timestamps.
   This fix is correct because steady_clock is monotonic and suitable for measuring time intervals.
   





