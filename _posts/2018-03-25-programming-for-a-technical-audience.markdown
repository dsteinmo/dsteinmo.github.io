---
layout: post
title:  "Developing Code for a Technical Audience"
date:   2018-03-25 13:10:33 -0500
categories: blog update
---

Most novice programmers tend to write code for an audience of one - themselves. They often write unformatted and/or unreadable code that is indecipherable by anyone else. This is how I started out writing my `MATLAB` and python scripts in grad school, and much of my code that's left over from those days remains in this state.

As programmers mature, they tend to expose themselves to code that is not their own. Learning to read other developers' code is a learning curve in and of itself. At some point, especially if they are writing code in the software industry, developers are told to "write code to be readable by other developers." Although this is a nebulous statement on its own, it opens up doors to conversations around proper formatting, indentation, naming conventions, and so on. Rules and best practices begin to emerge for enforcing and invoking these rules. Hopefully the rules are also backed by some kind of automation, e.g., linters and formatting utilities, where possible. Other best practices, like naming variables and functions such that the code is as self-documenting as possible, are enforced on an honour-system basis or by a code review process. 

It all sounds great, and it is. However, every rule has exceptions. I had to learn about such an exception the hard way in the case of developing for a technical audience. Let's delve deeper into the statement "write code to be readable by other developers." This statement, although very broadly true for the software industry, fails miserably outside of the software industry. Why? Because some code needs to be read by (non-software) engineers, data scientists, mathematicians, and other technical professionals/academics.

"So, what?" you might ask. All of the same readability and self-documentation rules apply equally well to non-developers, right? Wrong. Most of these non-developer individuals will only know a small subset of software development principles. This doesn't make them dumb, it just makes them not quite a software developer, yet. Showing them a verbose self-documenting code snippet makes them shut down due to word exhaustion. Showing them a beautifully designed object-oriented code package with a multi-file and multi-folder structure makes them shy away due to too much "clicking through files." Remember, most of these folks will not have a modern IDE installed for the programming language you're writing in, and they don't want to be bothered with downloading one. What may seem like simple code navigation to you with standard tricks like "go to implementation/declaration", "find all usages", and "show call hierarchy", might look like finding a needle-in-a-haystack to them. So what's the answer?

## Know your audience

The golden rule of "write code to be readable by other developers" should be adjusted to "write code to be readable by your audience." This is only a slight modification, but it warrants further discussion.

Is your audience a desktop user-interface end-user? Are they a developer? Are they a technical person with limited programming skills? The answer to the question of "who is my audience for this particular file/snippet/line of code?" may drastically change how you should write it, and it may make or break the marketability and usability of your software. In some sense, writing code needs to be thought of as analogous to providing a software API, where the consumer of your API changes depending on what the purpose of your particular code-chunk serves.

## An auto-biographical example

In early 2016, I came up with the idea of using some of my newly acquired industrial development skills to write some open-source education-oriented code for an academic audience. Should be a piece of cake, right? Well... it turns out, not so much.

Since python was, and still is, being sold as the *de facto* language for scientific computing purposes I decided to use it to write the project. No mistakes so far, and the project was planned to follow two development milestones:

	1. A simple python script to solve the shallow water equations as a proof-of-concept and as a starting point of reference.
	2. Re-write the shallow water solver using proper object-oriented design heuristics and other "best practices" from industry.

Step 1 went off without a hitch, but step 2 was where I went horribly wrong. Consider the following two implementations of roughly the same area of the code (see full implementation on [github](https://github.com/dsteinmo/pysws)):

### Procedural (single-script) implementation

{% highlight Python %}
	# ... (a portion of the main time-stepping loop)

        flux_qx[:,:,0] = q[:,:,1]                               
        flux_qx[:,:,1] = q[:,:,1]*q[:,:,1]/q[:,:,0] + 0.5*G*q[:,:,0]*q[:,:,0]
        flux_qx[:,:,2] = q[:,:,1]*q[:,:,2]/q[:,:,0] 
        
        flux_qy[:,:,0] = q[:,:,2]                        
        flux_qy[:,:,1] = q[:,:,1]*q[:,:,2]/q[:,:,0]
        flux_qy[:,:,2] = q[:,:,2]*q[:,:,2]/q[:,:,0] + 0.5*G*q[:,:,0]*q[:,:,0] 
           
        source[:,:,0] = np.zeros([NY, NX])
        source[:,:,1] = G*q[:,:,0]*background_depth_xderiv - F*q[:,:,2]
        source[:,:,2] = G*q[:,:,0]*background_depth_yderiv + F*q[:,:,1]
                       
        for i in range(0, NUM_FIELDS):
            # Compute divergence of flux (axis=1 is x, axis=0 is y).
            div_q[:,:,i] = np.real(ifft((1.j*k)*fft(flux_qx[:,:,i], axis=1), axis=1)) +  np.real(ifft((1.j*l)*fft(flux_qy[:,:,i], axis=0), axis=0))  
        
        # Compute RHS of PDE (LHS contains only dq/dt).    
        rhs_q = -div_q + source

        # Compute Runge-Kutta residual.
        res_q = RK4A[intrk]*res_q + dt*rhs_q

        # Update fields.
        q += RK4B[intrk]*res_q
{% endhighlight %}

### Object-oriented (multiple source files) implementation

{% highlight Python %}

	# ... (the only method in an LSERK4 time-stepper class)
    def step(self, solver, dt):
        rk4a = Lserk4TimeStepper.RK4A
        rk4b = Lserk4TimeStepper.RK4B
        res_q = 0.*solver.q

        for intrk in Lserk4TimeStepper.LSERK4_STAGES:
            rhs_q = solver.compute_rhs()
            res_q = rk4a[intrk]*res_q + dt*rhs_q
            solver.q += rk4b[intrk]*res_q

{% endhighlight %}

To the professional software developers out there - we win, right? We used ideas like information encapsulation and abstraction. We've abstracted complexity away from the reader, and our implementation is much shorter. This is a clear win, right? Wrong!

The problem with writing the code in this way is we've hidden the information our audience cares about away from them. Who cares about shallow water solvers? Mathematicans mostly, and some oceanographers, perhaps. A mathematician looking at this code out of context would say "where's the rest of it? Where is the spatial dicretization of the partial differential equation, and where is the "main" body that moves things forward a number of steps instead of just a single step?"

The answer to these questions again exhaust and probably confuse a non-developer: "Well, `step` is a method of a `TimeStepper` class.", "A `solver` is a class that has both an instance of a `TimeStepper` object and a `SpatialDiscretization` object.", "The spatial discretization only appears implicitly in the `solver.compute_rhs()` line which hides all the implementation details.", "The entire process is moved along by the `solver` object's `solve` method," which though aptly named and self-documenting, is painfully redundant. The problem is that we've taken away from our audience what they care about, and introduced a series of hoops to jump through, which they don't care about.

What did I learn from this process? That milestone 1 better served the educational-oriented requirement of the repository than milestone 2. Development on milestone 2 was naturally abandoned, and the two code-lines sit side-by-side on the `master` branch (though in different folders), as a quasi-completed/dead project.

## Conclusions

Open-source projects need to pay special attention to who the audience consuming their code is. At the end of the day, code is a product itself and needs to be treated as such. This is not to say that all code needs to appear in a single flat file with no object-oriented practices if you're writing for mathematicians. The trick is to subdivide the project into a _front-end_ or _API_ component and a _back-end_ or _library_ component.

The back-end should be written for developers, with occasional excerpts for the front-end audience if the code gets too specific to the audience's area of specialization. The _front-end_ ought to be readable, usable, and even re-writable in a style that will work for the particular audience. One prominent example is that mathematicians want to write code that reads like math. Why is MATLAB still so popular even though software packages like NumPy and SciPy exist and are free? It's because MATLAB syntax reads like mathematical notation, whereas python reads like computer science notation, and adding object-oriented semantics to the code only further burdens the audience to know more than they should need to know! 

Remember that MATLAB started out in the 1980s as a simple command-line interface wired up to some linear algebra libraries. Why was it so successful? Because it put the abstraction exactly where it needed to be by hiding the complicated library calls from the end-user (a mathematician or an engineer), and it gave them a syntax and interface that was almost natural to them. Moreover, they implemented many of the library methods that they had exposed to their users in the very syntax they wanted their end-user to know, i.e., they ate their own dog-food. As a result, the pieces of source code that were *open* to their users was readable in a language they already knew (or were going to learn anyways). 

Learning by example from where my project failed, and where a flagship like MATLAB succeeded was invaluable, and I hope that others can learn from the experience too!