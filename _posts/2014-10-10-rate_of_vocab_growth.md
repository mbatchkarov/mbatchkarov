---
layout: post
title:  "An exploration of scipy sparse matrices"
date:   2014-10-10 14:52:58
tags: [python]
---

My colleague [Matti Lyra](https://github.com/mattilyra) recently faced an
interesting computational problem.  He wanted to see how quickly a stream of
temporaly-ordered documents evolves, and he chose to do it by looking at how
often new words appear in the steam. This post is about how to do this
efficiently in Python.

More formally, the problem is: given a [term-document
matrix](http://en.wikipedia.org/wiki/Document-term_matrix) (stored as a
`scipy.sparse.csc_matrix`), find how many new terms each document introduces as
efficiently as possible. Here is what we came up with.

First let's generate some plausibly-looking (sparse) matrix to play with.


    import numpy as np
    from scipy.sparse import csc_matrix

    np.random.seed = 0
    mat = csc_matrix(np.random.rand(10, 12)>0.7, dtype=int)
    mat[1, 0] = 2 # add some variety to the matrix
    mat[0, 1] = 3
    print(mat.A)

    [[0 3 1 1 1 0 0 0 1 0 1 0]
     [2 0 1 1 0 0 0 0 0 1 1 1]
     [1 0 0 0 0 1 1 1 0 0 0 0]
     [0 1 0 0 1 0 0 0 0 1 1 0]
     [1 1 0 0 1 0 1 0 0 0 0 1]
     [0 0 1 0 0 0 1 0 0 0 0 0]
     [0 0 1 0 0 0 1 0 1 1 0 1]
     [0 1 0 1 0 1 0 0 0 0 1 0]
     [0 0 0 1 0 0 1 0 1 0 1 1]
     [0 0 0 0 0 0 1 0 1 1 0 1]]


The (sparse) matrix is stored as three dense arrays, `data`, `indices` and
`indptr`. `data` contains the non-zero values of the matrix, in the order in
which they would be encountered if we walked along the columns top to bottom and
left to right. If this were a `csr` matrix, the walk would have been along the
rows.


    mat.data[:10]
    array([2, 1, 1, 3, 1, 1, 1, 1, 1, 1])



These numbers correspond to the non-zero values we would encounter if we walked
along the columns of the matrix. Note column boundaries are not marked- this is
what `indptr` is for. `indptr[i]` tell us where the `i`-th columns begins. For
the matrix above, we have


    mat.indptr[:10] # start and end index of each column in the data array
    array([ 0,  3,  7, 11, 15, 18, 20, 26, 27, 31], dtype=int32)



So the first column starts at position `0` and runs until position `3`
(exclusive), the second columns starts at position `3` and runs until position
`7`, etc. Here are the non-zero values stored in the first and second column.


    print(mat.data[mat.indptr[0]:mat.indptr[1]])
    print(mat.data[mat.indptr[1]:mat.indptr[2]])

    [2 1 1]
    [3 1 1 1]


What is still missing is what row these values should go in. This is stored in
`mat.indices`. We can index it similarty to how we index `mat.indptr`. To get
the rows in the first column where the non-zero values go, we would do this:


    print(mat.indices[:10]) # position of the data in a given row
    print(mat.indices[mat.indptr[0]:mat.indptr[1]])

    [1 2 4 0 3 4 7 0 1 5]
    [1 2 4]


i.e. the non-zero rows in the first column are at positions 1, 2 and 4.

Let's get back to our original problem of finding how many new terms appear in
each document. First, we want to find when (in what document) a term first
appears, i.e. the position of the first non-zero row in each column.
Fortunately, the `scipy.sparse.csc` format makes that very easy:


    first_nonzero_row = mat.indices[mat.indptr[:-1]]
    first_nonzero_row
    array([1, 0, 0, 0, 0, 2, 2, 2, 0, 1, 0, 1], dtype=int32)



Now that we know when words first appear, we want to find how many words appear
for the first time in each document:


    new_cols_per_row = np.bincount(first_nonzero_row, minlength=mat.shape[0])
    print(new_cols_per_row)
    [6 3 3 0 0 0 0 0 0 0]


And there you have it- `new_cols_per_row[i]` corresponds to how many new terms
appeared in the `i`-th document.

One trick we missed in our original implementation is the `minlength` parameter
to `bincount`. Without it `numpy` will get rid of bins (i.e. rows) with zero new
features, and the result will be smaller than what we would expect:


    np.bincount(first_nonzero_row)
    array([6, 3, 3])



In the real text streams Matti works with, we didn't find this to be a problem,
i.e. every document had at least one new term.
