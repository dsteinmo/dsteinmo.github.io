---
layout: post
title:  "A new blog and a C++ project in its infancy"
date:   2017-12-26 06:59:15 -0500
categories: blitzdg update
---
I recently started a project written in C++ called blitzdg. It's a spin-off from my PhD work on discontinuous Galerkin (dg) methods and my goal is to do it *the right way* with MPI for distributed parallelism. 

I'm learning a lot about the various academic C libraries out there like `METIS` for mesh partitioning and `UMFPACK` for sparse matrix solvers. I'm also learning about the power of C++ that is unleashed by `boost` which adds loads of functionality!

My industrial work experience has familiarized me with build automation tools and continuous integration (CI) platforms. `blitzdg` uses [travis-ci](https://travis-ci.org/dsteinmo/blitzdg) for CI, and good old-fashioned GNU Make for its build. I've also integrated [coveralls](https://coveralls.io/github/dsteinmo/blitzdg) for tracking the percentage of code covered by the unit tests as the project matures. 

I won't go too deep into the project's code right now, but as a starting point, here's a call out to the `UMFPACK` C library for factorizing a compressed sparse matrix with pre-computed numeric factors LU.

{% highlight C++ %}
// ...
cout << "Solving Ax = b.." << endl;
umfpack_di_solve (UMFPACK_A, Ap, Ai, Ax, x, b, Numeric, null, null);
cout << "Done." << endl;
{% endhighlight %}

Check out the project's repoitory at [https://github.com/dsteinmo/blitzdg/](https://github.com/dsteinmo/blitzdg/).
