Concurrent Programming
======================

## History

- First Mutual Exclusion Problem and Solution
  - one CPU and one core
  - N processes share a memory and can be scheduled anytime
  - all have critical sections
  - "Solution of a Problem in Concurrent Programming Control", Dijkstra (1965)
    - <https://en.wikipedia.org/wiki/Dekker%27s_algorithm>
- Mutexes and Semaphores
  - "Cooperating sequential processes", Dijkstra (1965)
  - A semaphore is a special integer
    - atomic with a hidden wait queue
    - manipulated with up/down methods
  - A binary semaphore can be used as a mutex to protect critical sections
  - A binary semaphore can be used for signaling
    - A process down(&idle) and kicks up a worker
    - the worker finishes the work and up(&idle)
    - this use is undesirable in modern mutexes as it is error-prone
  - producer/consumer with unbounded buffer
    - producer: while (true)
      - { produce(); down(&mutex); push(); up(&mutex); up(&count); }
    - consumer: while (true)
      - { down(&count); down(&mutex); pop(); up(&mutex); consume(); }
  - rewrite to only use binary semaphores
    - producer: while (true)
      - { produce(); down(&mutex); push(); if (count == 1) up(&ready); up(&mutex); }
    - consumer: while (true)
      - { if (wait) down(&ready); down(&mutex); pop(); wait = (count == 0); up(&mutex); consume(); }
    - count is no longer a semaphore, but a state used by push and pop
- modern rewrite of producer/consumer with unbounded buffer
  - for signaling, semaphores have persistent signals and support both
    wait-before-signal and signal-before-wait
  - condition variables only support wait-before-signal
  - producer: while (true)
    - { produce(); down(&mutex); push(); if (count == 1) signal(&cv); up(&mutex); }
  - consumer: while (true)
    - { down(&mutex); if (count == 0) { wait(&cv, &mutex); } pop(); up(&mutex); consume(); }
