# sample-code

This page provides a brief overview for some of my more interesting historic work and deep links into the associated repos.

# Tag based invalidation in Memcached

When developing the caching layer for a customer CMS, Chronicle, we needed a shared high speed cache. We selected memcached. Unfortunately we also need to do tagged based invalidation (e.g. invalidate all 'project' related cache entries). Memcached only supports explicit identifier based invalidation. I designed and developed a Zend Framework cache backend to simulate this functionality in a performant manner.

The code includes comments covering the approach in more depth and can be found here:
https://github.com/gnicol/chronicle/blob/master/library/P4Cms/Cache/Backend/MemcachedTagged.php

# Custom Perforce Object Model

When working on Swarm, a code review web app for Perforce, we wanted to avoid running low level Perforce commands and instead model the various items/concepts as objects that are simpler to interact with. This involved some challening mappings as we had to determine which command(s) can be clumped into logical objects. We also had to support a broad range of versions for the Perforce server, making them appear identical to end consumers. A good deal of thought was put into code reuse; note the SingularAbstract and PluralAbstract for the specs. The end result is an easy to use, performant and very well tested (>90% code coverage) interface. I came up with the design and performed all of the initial implementation.

The spec models provide a nice example of well formatted code demonstrating good code reuse:
https://github.com/gnicol/swarm/tree/master/library/P4/Spec

# Read/Write Git Repo Replication

In our extended version of GitLab we needed to treat a remote git repo as the authorative master. This was a complex challenge. We needed to fully understand how Git behaves under high concurrency to ensure there as no chance of the repositories descynchronizing. Ultimately, I devised an approach relying primarily on git hooks and a custom 'receive-pack' wrapper.

At a high level, we wanted to read from the remote master before serving users data or accepting a push to ensure our local non-authorative repo was up to date. This half of the problem was fairly straight forward.

We also needed to accept write operations and forward them to the remote master, verify the remote master was satisfied, and only then accept the update. Its key to keep all locks as briefly held as possible, and never held during client network transfer (to avoid denial of service attacks). That presents quite a challenge to do in a manner that cannot deadlock. Ideally, a lock would be taken in the middle of the pre-receive hook and then released at the start of post-receive but these hooks are discrete processes.

To solve this, I designed a custom 'receive-pack' wrapper which listens on a unix socket and can be commanded by the pre/post-receive hooks to take or release the lock. As the receive-pack process lives through the full pre-reive and post-receive life cycle it can hold the lock for the required duration. As recieve-pack terminates when the push is completed or canceled, we can be assured the lock will not be held indefinitely. 

The locking strategy is still a bit broad, locking at the repository level not the reference/branch/tag level. In this instance the remote system had the same constraint so solving the problem in a more fine grained manner was skipped to maintain simplicity.

Internally, this is all documented in notably more detail but regretfally I cannot share all of that. You can however view the low level mirroring logic here: 
https://github.com/gnicol/gitswarm-shell/blob/release/perforce_swarm/mirror.rb

As well as the custom receive-pack script and lock handling here:
https://github.com/gnicol/gitswarm-shell/blob/release/perforce_swarm/bin/swarm-receive-pack
https://github.com/gnicol/gitswarm-shell/blob/release/perforce_swarm/mirror_lock_socket.rb
