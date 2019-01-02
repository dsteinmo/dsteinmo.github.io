---
layout: post
title:  "blitzdg turns one. Adds support for 2D triangular meshes."
date:   2019-01-01 13:37:00 -0500
categories: blog update
---

It's hard to believe that we ([the WQCG](https://wqcg.ca)) has been working on [blitzdg](https://github.com/WQCG/blitzdg) for over a year now. Now, this has not been a full year of "work days" in the 8-hour sense, but it's been a year of dedicated work nonetheless, and we have come a long way. In this post, I'll delve into what we've done, what we've learned, what it all means, and what's next.

## What we've done

Although it didn't take too long to get one-dimensional equation solvers working within the little C++ world we created, it was a much bigger lift to get something that we could show off in 2D. There were times of frustration, even exhaustion, but in the end we have reached a milestone that we can be proud of - a working two-dimensional shallow water solver that (in principle) can work in arbitrary geometries.

The short-list to get us to where we are goes as follows:

- 1D wave solvers (advection and burgers equations).
- GMRES linear iterative solver, and proof-of-concept 1D elliptic solver (Poisson's equation).
- Triangular grid representation in C++; Gmsh .geo file parsing.
- Explicit nodal points construction and orthogonal polynomials on the triangle.
- 2D generalized Vandermonde matrices; Surface integral lifting operator.
- Mapping to standard triangle, with geometric factor calculation.
- Volume-to-Face masks; boundary node masks.
- 2D 4th-Order Runge-Kutta Shallow Water solver.

It may not sound like a lot, but take into account the fact that we've maintained 96% code coverage with BDD-style unit tests. Add to that all the hours lost to debugging silly transcription errors and memory-reference errors, trust me, it is a lot.

## What we've learned

#### Cross-platform native development is hard.

We shouldn't have to explain this one too hard, as it kind of goes without saying. Many of the scientific packages we rely on don't have MSVC builds on Windows, and their mingw support leaves something to be desires. Initially, `blitzdg` had two CI processes - TravisCI for Linux and AppVeyor for Windows.

Inevitably, it became an exercise in hair pulling to get builds to pass both CI processes in unison before a merge to master. Meanwhile, CircleCI took a huge leap forward with their docker container builds that allowed for a lot of dependency-caching, it was only natural to move over to Circle. This happened in two phases. First the Travis/Linux build was moved into a Circle container. Second, we killed off the windows build by making the choice to cross-compile to mingw from within a linux container. It wasn't particularly pretty, but in the end we've pulled it off and will only need to mess with it when we add or kill dependencies.

#### Even if a library is abstracting away the contiguous memory representation of your 2D/3D data structures, you should still think about the memory-block representation of your arrays.

Yeah, yeah, `blitz` is great and everything, but that doesn't mean you completely avoid cache-thrashing and performance hits for free. 

2D and 3D contiguous arrays just don't exist in C/C++. C arrays are row-major, Fortran arrays are column-major. You have to keep this in mind when making calls between the languages. Also, I've learned that the nesting loops in the wrong order can cause huge problems. Always think about what your array actually looks like in memory, and when you might be missing the cache. If it's possible to make a cache-miss, then you are probably doing it.

#### Keep up that test coverage, you'll thank yourself later.

It's too easy to say "yeah, I'm pretty sure this is right." Or, "Something else will tell me when this is broken." It saves time and headaches in the long run to write behaviour-driven test clauses with proper assertions. I know you've probably heard it before, but it's worth saying again.

You may also be drawn to omit tests on the grounds that, "what I'm writing is small, and not worthy of a test." However, if it's worth writing code for it, then it must be some piece of a larger puzzle, and without tests that assert correctness, you leave room for skeletons to emerge from the closet later down the development road. Do yourself a favour, and fully test (and assert the correctness of) your APIs!

## What it all means

Well, it means hopefully we're approaching a state where we become useful to somebody else, or even ourselves as scientists who want to consult on state-of-the-art numerics.

As I learned in grad school, it's a matter of sales savvy, confidence, and real-world data absorption to turn a toy code into an operational scientific model. We can do all these things, and we fully intend to.

## What's next?

We're still thinking about what the priority stack for 2019 looks like, but here's a smattering of ideas that are outstanding and we would like to do in 2019.

- Quadrilateral elements and nodes, and unstructured quad grids.
- Kalman filtering tools for data assimilation.
- Curvilinear elements.
- NetCDF/Vtk output classes.
- Incompressible Euler/Navier-Stokes equations solver in 2D, with modern DG stabilization techniques.
- 3D "layered" elements (i.e., structured in one of the coordinates). 
- 3D solvers and models for lakes.

Want to help? Want a feature? Want to bump something up the priority stack? Join the conversation on our [slack workspace](https://wqcg.slack.com). We'd love to hear from you and what scientific problems you're working on!
