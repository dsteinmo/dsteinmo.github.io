---
layout: post
title:  "Yo dawg, I heard u like Fortran"
date:   2018-01-27 22:30:33 -0500
categories: blog update
---
From my grad school days, I had heard that it was a common practice to mix C/C++ code with Fortran libraries such as `LAPACK` or `BLAS`. So I set about trying to leverage `LAPACK`'s eigenvalue solver routines for `blitzdg`'s EigenSolver class.

The process wasn't overly challenging, but there were some intracies that needed special attention. I'll spell those out here in hopes that it may help someone in the future, or that it will serve as a reference for myself.

### Declaring an external Fortran routine as a C/C++ function

In my case, I want to call the Fortran routine called `dsyevd`, which is a work-horse for solving eigenvalue problems of real, symmetric matrices. Its official API doc is [here](http://www.netlib.org/lapack/explore-html/d2/d8a/group__double_s_yeigen_ga77dfa610458b6c9bd7db52533bfd53a1.html#ga77dfa610458b6c9bd7db52533bfd53a1).

In order to call `dsyevd` in my C++ code, I need to declare a function with the same signature as the Fortran routine, but it's a bit more convoluted than that. Here is what the declaration code ends up looking like:

{% highlight C++ %}
extern "C" {
    void dsyevd_( char* jobz, char* uplo, int* n, double* a, int* lda,
                double* w, double* work, int* lwork, int* iwork, int* liwork, int* info );
}
{% endhighlight %}

Questions you might have:

1. Why is it wrapped in an `extern "C" {}` block?

This tells the compiler to give the funciton C-style linkage. C++ compilers will often mangle the name. This is bad, because we need the function name to match up with the Fortran routine, otherwise the linker will not associate the two pieces of code.

2. Why is there an underscore after the function's name?

Fortran compilers like `gfortran` will usually append an underscore to the routine's name in the object file. There is a good discussion on Fortran/C++ interoperability [here](http://www.yolinux.com/TUTORIALS/LinuxTutorialMixingFortranAndC.html). Thus, in order for the linker to find the code associated with our function, we append an underscore to the name.

3. Why are all the arguments pointers?

This is a difference between how C/C++ and Fortran compilers work. C/C++ passes arguments by value. Fortran always passes arguments by reference. Therefore, the C++ declaration of the Fortran routine contains a pointer in every argument.

### Calling a LAPACK routine from C/C++ code

The next step is to actually call the C function we declared which links to a Fortran routine in a C++ method. The code looks like this: 

{% highlight C++ %}
/**
 * Solve Ax=λx using LAPACK. Eigenvalues are stored in reference 'eigenvalues' and eigenvectors are stored column-wise
 * in reference 'eigenvectors.'
 */
void EigenSolver::solve(const Array<double,2> & A, Array<double,1> & eigenvalues, Array<double, 2> & eigenvectors) {

    int sz = A.rows();
    int lda = sz;
    int iwkopt;

    double ww[sz];
    double wkopt;
    int lwork = -1;
    int liwork = -1;
    int info;

    char JOBZ = 'V';
    char UPLO[] = "UP";

    double * Apod = new double[sz*lda];

    MatrixConverter.fullToPodArray(A, Apod);

    /* Determining optimal workspace parameters */
    dsyevd_( &JOBZ, UPLO, &sz, Apod, &lda, ww, &wkopt, &lwork, &iwkopt, &liwork, &info );

    lwork = (int)wkopt;
    double * work = new double[lwork];
    liwork = iwkopt;
    int * iwork = new int[liwork];

    /* Solve eigenproblem */
    dsyevd_( &JOBZ, UPLO, &sz, Apod, &lda, ww, work, &lwork, iwork, &liwork, &info );

    MatrixConverter.podArrayToFull(Apod, eigenvectors);

    for (int i=0; i < sz; i++)
        eigenvalues(i) = ww[i];
}
{% endhighlight %}

First, we set some parameters and convert the blitz matrix `A` to a contiguous memory block (a 1D array) in `MatrixConverter.fullToPodArray`. 

The `dsyevd_` function then gets called twice. First to determine the optimal workspace parameters, then to actually solve the eigenvalue problem. The Fortran routine stores the resulting eigenvectors in the storage allocated by the array `Apod` (Here, `pod` = plain-old data-type). 

The eigenvectors are then converted from `pod` format to 2D blitz-array format, and similarly the eigenvalues are put into the 1D blitz array that the caller passes in.