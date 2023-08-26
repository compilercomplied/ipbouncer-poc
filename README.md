# IP bouncer service

_Disclaimer: this is the take home exercise from a past final interview. I was ghosted and no feedback was given during the interview, after following a long process. I've reworded the task description to make it very hard to find when searching on the internet._

# Service description

> We want to provide the possibility of blocking traffic from untrusted IP addresses to our systems. We’ll use [this source](https://github.com/firehol/blocklist-ipsets) as a reference for now. This service will be executed in a hot path (authentication pipeline), so performance is critical. We want to avoid any network latency overhead so for our first iteration we'll avoid relying on an external data store.
>
> In the linked source, there are many lists available, but we’ll focus on starting with only the aggregate lists: `firehol_level1` through `firehol_level4`. In the future more lists may be added.
>
> Lists get updated frequently and we want to avoid having to manually update them.
>
> We’d also like to return any possible relevant metadata when the IP is blocked.

# Requirements

## Functional

1. Check whether an IP is blocked or not and retrieve metadata about it.
2. Update blocklists automatically.
   1. Neither a frequency nor a load target has been given. Using the four lists given as a reference we can infer 1 minute check frequency (\*) at most and small updates of a few hundred IPs, with scattered updates of dozens of thousands.

(\*) 1 minute check frequency is the update frequency for the given lists.

## Non-functional

1. Needs to be fast. It is being executed in a critical hot path (authentication pipeline).
2. Deployed solution can be monitored properly.
3. General expectations of reliability, consistency, availability and cost.

# Data analysis

## Introduction

Here we’ll be talking over general expectations derived from the data; its volume, shape and how it mutates throughout time. Using this, we’ll define strategies for storing it, keeping it up to date and searching through it.

We want to keep it as fast as possible so trying to keep everything in memory seems a reasonable approach. If we go this route, we also have to keep in mind how quickly can we spin a new process, since ingesting the data from the lists is what will dictate whether the process is ready to receive traffic or not.

The only metadata I can infer from the given spec comes from the list itself and not from the IPs, so we can safely assume metadata has a small enough storage footprint to disregard it in these estimations; we can simply pass around references to this static list-wide data.

### Data shape

We are working with 4 files containing ordered sets of both IPs and IPs with subnet masks. Each file has a header with some easy to parse metadata. The lists have around 300K lines worth of a mix of IPs and subnets.

Since each source represents a different taxonomy of threats, we need to keep track of each list individually if we want to provide that related metadata to consumers of the service.

### Data flow

We’re lucky enough to have a partial history of changes so we can better understand how the data mutates both in frequency and volume.

![firehol_level1 changes history](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/Untitled.png)

firehol_level1 changes history

![firehol_level3 changes history](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/Untitled%201.png)

firehol_level3 changes history

![firehol_level2 changes history](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/Untitled%202.png)

firehol_level2 changes history

![firehol_level4 changes history](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/Untitled%203.png)

firehol_level4 changes history

We’re safe assuming average update of a few thousand unique IPs at most with some outliers in the hundreds of thousands.

Even though our scenario is read heavy, we have occasional spikes with relatively big updates, and it is something that needs to be considered for our test suites.

I’ve manually checked how updates are pushed to some of the lists, and while the unique IP add/del counts might spike, the actual data changed in the file is probably below the hundreds; removing a single subnet might remove thousands of unique IPs, but ultimately you’re only removing a single item from the list.

Regarding volume, following graphics don’t show sudden spikes in quantity of registered IPs.

![firehol_level1 volume history](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/Untitled%204.png)

firehol_level1 volume history

![firehol_level3 volume history](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/Untitled%205.png)

firehol_level3 volume history

![firehol_level2 volume history](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/Untitled%206.png)

firehol_level2 volume history

![firehol_level4 volume history](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/Untitled%207.png)

firehol_level4 volume history

All of this should be double checked with someone that has a better understanding of the domain than myself.

## Brute force approach - hash map

### Description

Since IPs are unique we can simply use them as keys in a hash map and retrieve the relevant metadata like the list to which the IP belongs. A miss implies the IP is not filtered.

Before doing this we need to pre-generate all the available permutations in the lists.

### Analysis

If we generate all the permutations derived from the 4 lists we get over six hundred million IPs. Each IP can be represented using 4 bytes, so if we were to generate all the permutations for available IPs we’d end up with roughly **2.5GB** worth of data. Small enough to keep it cached in memory even for the future possible additions.

Since I wasn’t sure how much time would it take to loop over ~600M items I drafted a script to get a rough understanding of its feasibility.

- Some code

  ```rust
  use std::{collections::HashMap, time::Instant};

  use rand::Rng;

  fn main() {
      let arr: Vec<i32> = vec![0; 700000000];
      let mut hm = HashMap::new();
      let mut rng = rand::thread_rng();

      let now = Instant::now();
      for i in arr {
          // Add some noise so the compiler doesn't trick us into some
          // optimization.
          let dynamic_value: u8 = rng.gen();
          hm.insert(i.to_string(), dynamic_value.to_string());
      }
      let elapsed = now.elapsed();

      println!("elapsed iter time: {}ms", elapsed.as_millis());
  }
  ```

  ![Untitled](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/Untitled%208.png)

I’ve executed this through WSL2 on my desktop so it’s not really a good measure at all (not even sure what my CPU is right now) but at least we have some ground to talk about times. It took 43 seconds to push 700M items to a hash map.

I don’t want to waste too much time hunting for accurate numbers so lets do some magic and assume that both generating all permutations and persisting them is going to double this time, which leaves us with one and a half minutes of additional “build time” for the API component. Undesirable, but it could be acceptable.

### Summary

We can see how the problem space explodes as we try to handle all the possible permutations.

1. ✅ Simple enough to develop, no logic aside from the straightforward operation of calculating IP ranges from subnets.
2. ✅ Easy and fast to update; both insertion and deletion have constant time complexity and are trivial to deal with.
3. ℹ️ While a hash map is considered to be O(1) access/insert/delete in theory, with this size we could be hitting scenarios in which updates an lookups takes longer than expected
4. ❌ Requires _normalizing_ all IPs with subnet masks, incurring in extra overhead.
5. ❌ Extra work when pushing updates; if we remove a subnet and add a different one we’re now working in the dozens of thousands of changed items instead of simply changing the reference to the subnet.

## Range based solution - interval tree

### Description

Subnets are scoped to their relevant octets, so we can simplify our problem space by simply building trees that contain ranges. The proposed implementation is not an interval tree but it helps me signal that nodes will check for ranges.

Trees will have at most 4 levels depth and can be quickly traversed.

![Example tree with random values selected from `firehol_level1.netset`.](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/example_trees.png)

Example tree with random values selected from `firehol_level1.netset`.

### Analysis

Calculating derived IP ranges from a subnet mask is a trivial task. When looking whether an specific IP is registered in the list we can simply check whether it is an exact match or falls in one of the ranges.

Let’s just go overboard and assume a couple things when giving an estimation of storage footprint:

1. Every line of the aggregated 300K lines has a subnet mask.
2. Building our trees ends up with every node containing a range (2 uint8 per node).

So, this gives us, _at most_:

- (300,000 subnets) _ (4 nodes per subnet) _ (2 digits per node) \* (4 bytes per uint8) = 9,600,000 bytes or **9.6mb total**.

Even if we factor in additional structures needed to glue everything together that number is not going to skyrocket.

### Summary

Our problem space is greatly reduced if we think about subnets instead of unique IPs. Let’s assume we won’t face

1. ✅ Calculate IP ranges from subnet masks is trivial and simple enough to implement.
2. ✅ Greatly reduced memory footprint, initial setup and change management.

## Conclusion

Reaching out to a tree based approach seems to be the implementation with less accidental complexity.

---

# Solution overview

## Description

If we disregard possible failures we can represent this as a single http application with a worker process pushing changes through a PATCH/PUT method. This should easily scale up to thousands of requests per second since we’re simply retrieving in memory data that has been structured to be easily searched.

Since we don’t live in a world without failures, we must build some redundancy to maintain an availability guarantee.

<aside>
ℹ️ I’ve assumed pushing changes asynchronously through a queue is a viable solution and that the delay on updates is negligible.

</aside>

We will need multiple components.

1. HTTP API to serve requests.
2. Worker that processes changes and pushes them to the API.
3. Queue or bus that helps pushing changes to API instances under a LB.
4. Internal repository to keep track of changes and decouple last known file contents from external sources. This way we don’t constraint spinning up new processes by a third party’s infrastructure.

### Alternative solutions

I’ve weighted in the idea of pushing changes directly to the API from the worker, removing the queue. This could be accomplished in two different ways.

1. Deploy both processes together + a different worker to pull changes from the external repo.
2. Keep track of deployed instances and iterate over them in the worker.

I don’t think neither can work due to their additional complexity, although the first one could do the trick if we really need near instant updates.

## C4 based overview

- C1 - context

  ![c1 - context.png](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/c1_-_context.png)

- C3 - components

  ![c3 - components.png](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/c3_-_components.png)

- C4 - modules (HTTP API)

  ![c4 - modules.drawio.png](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/c4_-_modules.drawio.png)

## HTTP API

The API is simple enough, a single endpoint to match by incoming IP. Its only unconventional aspect is the need to preload the data before being readily available. As previously mentioned, data should be retrieved from our internal repository to avoid dependencies with third parties.

![Application lifecycle](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/lifecycle.drawio.png)

Application lifecycle

### API draft

`GET /api/v1/blocks?ip=${string:required}`

<aside>
ℹ️ Retrieve whether an IP is blocked or not, with relevant additional metadata.

</aside>

- 200 - returned when a match is found.

  Example response:

  ```json
  {
    "metadata": {
      "blocklist": ""
    }
  }
  ```

    <aside>
    ℹ️ Additional fields can be added to the metadata object without a breaking change (i.e. last update known to that IP). More objects at the root can be added without incurring in a breaking change either.
    
    </aside>

- 204 - returned when there is no match.
  Example response (empty body):

  ```json

  ```

## Worker

We’ve previously defined an acceptable refresh frequency as 1 minute, so this flow should execute way under 60 seconds. I can’t foresee a reason why it would not, aside from possible latency issues.

![Execution flow](IP%20bouncer%20service%201d4f6b9758a64e92b024f1ad635df513/worker_flow.png)

Execution flow

## Infrastructure

The deployment model I’m used to is to simply throw to our PaaS managed k8s cluster the code and let it handle everything. Configuration of the cluster behavior is done through IaaS templates, and monitoring is handled by observability instrumentation that the PaaS provides.

Logs are shipped to elastic through official libraries/clients and indexed accordingly.

Ideally, we should be able to see a request throughout the whole component stack, and see aggregations of latency, failures, number of requests and also process’ resource consumption (memory, CPU, network bandwidth).

Alerts can be tuned to match SLIs.

All of this is cleanly provided by your average cloud vendor, with integrations to trigger alerts through webhooks included in the stack.

# Identified risks

1. **Stale data triggering false positives**. Hard to weight by myself, but I assume this as relatively innocuous.

# Identified challenges

## Horizontal scaling

This is a stateful service. It holds dynamic data that is routinely updated. Without any external storage service, be it a database or a distributed cache, we’re keeping dynamic data alongside the deployed process.

This lends to consistency issues when trying to scale horizontally, since each process owns its data. If deployed as multiple instances and abstracted away by a load balancer we now have two hard to tackle issues:

1. **Data consistency**. If we were to require multiple processes behind a load balancer, maintaining both states synchronized is necessary to avoid randomly triggering them.
2. **Updating the data**. If we let a different component handle the processing of updates, we can’t cleanly push them to deployed instances, since the purpose of having a load balancer is turning these `n` instances into a black box.

An alternative way of scaling horizontally can be segmenting traffic, which incurs in additional complexity. An example would be segmenting service deployments by region, which would also help reduce latency. Business branch, external vs internal, and similar splits come to mind that can further segment the traffic to help with load.
