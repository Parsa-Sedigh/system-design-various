https://www.youtube.com/watch?v=su3-fAEePs0&list=LL&index=1&t=7s&ab_channel=ByteMonk

## CI
- integrate code changes into the main branch of a shared source code repo
- automated testing
- build the code automatically

Once the code has been tested and built as part of the CI process, CD takes over.

## CD(continuous delivery)
Deploy the package to any env at anytime

- provision infra
- deploy app(to testing or production env)

## Functional requirements
Ask what are the functional requirements?

We're gonna building a simple CI/CD pipeline that takes the code, builds it into a binary and deploys it globally. We're not covering testing.

This system can be divided into two sub-systems:
- build system: builds our code into binaries
- deployment system

We can refer to specific version of the code through SHA which is a pointer to a specific commit.

- 5 regions
- 100,000 machines
- the deployment takes at least 30 mins per deployment
- 2-3 nines of availability

Jobs run in orderly matter. So first in first out. So we need to implement a queue to make sure that the order is maintained.

Pushing the commits trigger the builds.

The servers that process the builds are called **worker nodes**. After the build, we need to store and package them into an artifactory,
but let's say we're gonna use amazon s3(which is a blob store).

Since availability is important in our case, we need to ensure that we have some health check service to monitor the health of worker nodes.
So if they go down, we bring them up in some way and also they are auto-scalable.

We can represent the queue in a table(tabular format) where every job is a record in the table. It could be as simple as MySQL DB table.

The `last_seen` column is last seen heartbeat of the worker servers. The health check services send a heartbeat to worker nodes and store
the result to the last_seen column, making sure the last_seen time is not big enough to trigger an alarm that there is a problem with the servers.
So let's say the last_seen of any of the worker nodes responsible for this job is more than 30 mins, in that case, we can trigger an alarm
making sure the auto-scaling clicks in and increase the number of worker nodes.

This table will be used by 100s if not 1000s of workers and since we're using a mysql DB, it guarantees ACID transactions that will make it
safe for potentially all these worker nodes to grab jobs of the queue without unintentionally running the same job twice, so avoiding
any kind of race conditions.

The transaction will look like this:
```sql
begin transaction
select * from build_jobs where status = "queued"
order by created_at asc limit 1;
update build_jobs set status = "running" where id = ?
commit;
end transaction;
```
So a worker node(when it is available), will go to this table and look for a job with status "queued"(a job that is not running) and this query
will ensures that the job it will get, is the oldest job(first in first out) which is why we have `order by` and then once the worker node(s) picks
up the job, it will update the job record in the DB and set the status to "running" to ensure no other worker node picks it up.

We used a transaction to avoid race condition.

Let's estimate the minimum number of workers we would need to run our ci/cd system without any hiccups.

Let's say we have:
- 5000 builds/day
- each worker node can process up to 100 builds per day

So the total number of workers required is at least 50.

Note: Builds are not consistently done throughout the day. Because it might be the case that engineers are usually running the builds towards
the end of the day or in the beginning of the day. So the number of worker nodes required, will not be consistent throughout the day.

We should be able to **automatically** add or remove the number of workers whenever we need to. We can also scale vertically in some of the
hours of the day.

Once the worker node completes the build, we can store the binary into s3 before updating the relevant row in the table.
This ensures the binary has been persisted before it's relevant job has marked succeeded in our db.

Since we're going to deploy our binaries to machines spread across the world, we need a **regional storage** rather than just a single global
blob store and we can design our system based on regional clusters around the world like having 5 or 10 global regions and each region can have
a blob store(s3).

After placing the binary in s3, the blob store that gets it, will do asynchronous replication to store the binary in all the regional s3 buckets
This step should take no more than 5-10 minutes which brings our build and deploy duration to roughly 20-25 mins. So 15 minutes for build and 5-10
mins for global replication of the binary

Now our deployment system needs to allow for very fast distribution of all these 10gb binaries to hundreds if not thousands of machines across
all of our global regions.

Now we can update the jobs table with status succeed only when the binaries are replicated across all the regions because if we do it before that, and
the replication fails, the system can become inconsistent. So we'll also need an additional service that tells us that when a binary has been replicated
in all the regions. So this service continuously checks all the regional s3 buckets and aggregates the replication status for succesful builds.
In other words, it checks that a given binary in the main blob store has been replicated across all regions and once a binary has been
replicated across all regions, the service updates a separate sql db with the rows containing the name of the binary and a col for
the replication status and once the binary has a complete replication status, it is deployable.

For our blob distribution, since we're gonna deploy 10 gbs to 100s not 100s of machines, even with our regional clusters, having each machine
download a 10gb file one after the other from regional blob store, is going to be slow. So we need to use a peer to peer approach here which
will be much faster and will allow us to get to not more than 30 mins timeframe.

Let's describe what happens when an engineer clicks on the deploy btn to deploy the binary to every machine globally?

This is the action that triggers the binary downloads on all the regional peer to peer networks. Now to simplify this process and to support
having multiple builds getting deployed concurrently, we can design this in a **goal state oriented** manner. The goal state will be
desired build version at any point in time and this can be stored in a key-value store like zookeeper which will have a global goal state as well as
a regional goal state. So each regional cluster will have a key-value store that holds config for that cluster about what builds should
be running on that cluster and we will also have a global key-value store.

So now the engineer clicks the **deploy v1** button, our global kv store build version will get updated and the regional kv store will be
continuously polling the global kv store say every 10-15 seconds for updates to the build version and will update themselves accordingly.
Now machines in the cluster regions will be polling the relevant kv store and when the build version changes, they will try to fetch that build
from the peer-to-peer network and run the binary.