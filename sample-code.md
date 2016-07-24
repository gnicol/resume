# sample-code

This page provides an overview for some of my more interesting historic work and deep links into the associated repos.

# Tag based invalidation in Memcached

When developing the caching layer for a custom CMS, Chronicle, we needed a shared, scalable, high speed cache. We selected memcached. Unfortunately we also needed to do tagged based invalidation (e.g. invalidate all 'project' related cache entries). Memcached only supported explicit identifier based invalidation. I designed and developed a Zend Framework cache backend to simulate this functionality in a performant manner.

The code includes comments covering the approach in more depth and can be [found here](https://github.com/gnicol/chronicle/blob/master/library/P4Cms/Cache/Backend/MemcachedTagged.php).

# Custom Perforce Object Model

When working on Swarm, a code review web app for Perforce, we wanted to avoid running low level Perforce commands and instead model the various items/concepts as objects that are simpler to interact with. This involved some challenging mappings as we had to determine which command(s) can be clumped into logical objects. We also had to support a broad range of versions for the Perforce server, making them appear identical to end consumers. A good deal of thought was put into code reuse; note the [SingularAbstract](https://github.com/gnicol/swarm/blob/master/library/P4/Spec/SingularAbstract.php) and [PluralAbstract](https://github.com/gnicol/swarm/blob/master/library/P4/Spec/PluralAbstract.php) for the specs. The end result is an easy to use, performant and very well tested (~90% code coverage) interface. I came up with the design and performed all of the initial implementation.

The spec models provide a nice example of well formatted code demonstrating good code reuse [see here](https://github.com/gnicol/swarm/tree/master/library/P4/Spec).

# Bidirectional Git Repo Synchronization

In our extended version of GitLab, we had to adjust the local repos to act as a write-through proxy to an external system. This was a complex challenge. We needed to fully understand how Git behaves under high concurrency to ensure there was no chance of the repositories desynchronizing. Ultimately, I devised an approach relying primarily on git hooks and a custom 'receive-pack' wrapper.

At a high level, we had two problems. Firstly, we wanted to ensure the GitLab copy of the repo was up to date before serving any read or write operations. This half of the problem was fairly straight forward. We simply fetched from the remote repo before servicing the user’s pull/push request.

Dealing with write operations was more complex. There was the risk of simultaneous writes to both sides. There was also the challenge of ensuring the two systems, with varying opinions on what was valid, were both ok with the update.

Ultimately I decided to validate but not initially accept the push in GitLab. We then forward the push to the remote git repo which will either fully accept and commit it or reject it. Assuming that goes well, we accept the update locally.

This ended up up requiring locking to ensure another push didn’t come into GitLab in the middle of the validate/re-push/accept cycle leading to errors. It was key to keep all locks as briefly held as possible, and never held during client network transfer (to avoid denial of service attacks). That presents quite a challenge to do in a manner that cannot deadlock. Ideally, a lock would be taken in the middle of the pre-receive hook and then released at the start of post-receive but these hooks are discrete processes.

To solve this, I designed a 'receive-pack' wrapper which listens on a unix socket and can be commanded by the pre/post-receive hooks to take or release a local file system lock. As the receive-pack process lives through the full pre-reive and post-receive life cycle it can hold the lock for the required duration. Given recieve-pack terminates when the push is completed or canceled, we can be assured the lock will not be held indefinitely. 

The locking strategy is still a bit broad, locking at the repository level not the reference/branch/tag level. In this instance the remote system had the same constraint so solving the problem in a more finely grained manner was skipped to maintain simplicity.

Internally, this is all documented in notably more detail but regretfully the pages are confidential. You can however view the low level [mirroring logic here](https://github.com/gnicol/gitswarm-shell/blob/release/perforce_swarm/mirror.rb).

As well as the custom [receive-pack script](https://github.com/gnicol/gitswarm-shell/blob/release/perforce_swarm/bin/swarm-receive-pack) and [lock socket logic](https://github.com/gnicol/gitswarm-shell/blob/release/perforce_swarm/mirror_lock_socket.rb).
