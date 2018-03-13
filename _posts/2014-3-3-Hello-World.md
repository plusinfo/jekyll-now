---
layout: post
title: Allreduce (or MPI) vs. Parameter server approaches
---

In the last 7 years or so there has been quite a bit of work on parallel machine learning approaches, enough that I felt like a summary might be helpful both for myself and others. In each case, I put in the earliest known citation. If I missed something please comment.

One basic dividing line between parallel approaches is single-machine vs. multi-machine. Multi-machine approaches offer the potential for much greater improvements than single-machine approaches, but generally suffer from a lower bandwidth between components of the parallelized process.

Amongst single machine approaches, GPU-based learning is the dominant form of parallelism. For many algorithms, this can provide an easy 10x speedup, with the limits being programming (GPUs are special), the amount of GPU RAM (12GB for a K40), the bandwidth to the GPU interface, and your algorithms needing care as new architectures come out. I’m not sure who first started using GPUs for machine learning.

Another important characteristic of parallel approaches is deterministic vs. nondeterministic. When you run the same algorithm twice, do you always get the same result? Leon Bottou tells me that he thinks reproducibility is worth a factor of 2. I personally would rate it somewhat higher, just because debugging is such an intrinsic part of using machine learning algorithms and the debuggability of nondeterministic algorithms is greatly impaired.

MPI gradient aggregation (See here (2007).) Accumulate gradient statistics in parallel and use a good solver to find a good solution. There are two weaknesses here:
Batch solvers are slow compared to online gradient descent approaches, at least for the first pass.
Large datasets typically do not sit in MPI clusters. There are good reasons for this—MPI clusters are typically not designed for heavy data work.
Map-Reduce statistical query algorithms. The first paper (2007) of this sort was single machine, but it obviously applied to map-reduce clusters of the time starting the Mahout project. This addressed the second problem of the MPI approach, but not the first (batch algorithms are slow), and created a new problem (iteration and communication are slow in a map-reduce setting).
Parameter averaging. (see here (2010)). Use online learning algorithms and then average parameter values. This dealt with both of the drawbacks of the MPI approach as applied to convex learning, but is limited to convex(ish) approaches and may take a long time to converge on datasets where a second order optimization is needed. Iteration in a map-reduce paradigm remains awkward/slow in many cases.
Graph-based approaches. (see here (2010)). Learning algorithms that are represented by graphs can be partitioned across compute nodes and then operated on with parallel algorithms. This allows models larger than the state of a single machine. This addresses many learning algorithms that can be represented this way, but there are many that cannot be effectively represented this way as well.
Parameter server approaches. (see here (2010)). This is distinct from graph based approaches in that parameter storage and updating is broken out as a primary component. This allows models larger than the state of a single machine. Parameter server approaches require nondeterminism to be performant. There has been quite a bit of follow-up work on parameter server approaches including shockingly inefficient systems(2012) and more efficient systems(2014) although they remain less efficient than GPUs.
Allreduce approaches. (see here (2011)) Allreduce is an MPI-primitive which allows normal sequential code to work in parallel, implying very low programming overhead. This allows both parameter averaging, gradient aggregation, and iteration. The fundamental drawbacks are poor performance under misbalanced loads and difficulty with models that exceed working memory in size. A refined version of this approach has been used for speech recognition (2014).
GPU+MPI approaches. (see here (2013)) GPUs are good and MPI is good, so GPU+MPI should be good. It is good, although there are caveats related to the phenomenal amount of computation a GPU provides compared to the communication available, even with a good interconnect. See the speech recognition paper above for a remediation.
Most of these papers are about planting a flag rather than determining what the best approach to parallelization is. This makes determining how to parallelize learning algorithms rather unclear. My present approach remains case-based.

Don’t do it for the sake of parallelization. Have some other real goal in mind that justifies the effort. Parallelization is both simple and subtle which makes it unrewarding unless you really need it. Strongly consider the programming complexity of approaches if you decide to proceed.
If you are locked into a particular piece of hardware or cluster software, then you don’t have much choice—make the best of it. For some people this is an MPI cluster, Hadoop, or an SQL cluster.
If your data can easily be copied onto a single machine, then a GPU based approach seems like a good idea. GPU programming is nontrivial, but many people have done it at this point.
If your data is of a multimachine scale you must do some form of cluster parallelism.
Graph-based approaches can be the right answer when your graph is not too deeply interconnected.
Allreduce-based approaches appear effective and easy to use in many other cases. I wish every cluster came with an allreduce library.
If you are parsing limited (i.e. for linear representations) then a CPU cluster is fine.
If you are compute limited, then a cluster of GPUs seems the way to go.
The above leaves out parameter server approaches, which is controversial since a huge amount of effort has been invested in parameter server approaches and reasonable people can disagree. The caveats that matter for me are:

It might be that the right way to parallelize in a cluster has nothing to do with the right way to parallelize on a single machine, but this seems implausible.
Success/effort appears relatively low. It’s surprising that you can effectively compete with mature parameter server approaches on compute heavy tasks using a single GPU.
Note that I’m not claiming parameter servers are useless—I think they could be effective if applied in situations where you cannot prebalance the compute load of parallel elements. But the extent to which this applies in a datacenter computation seems both manageable and a flaw of the datacenter that will be reduced with time.
