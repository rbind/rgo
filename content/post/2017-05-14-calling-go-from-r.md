---
title: Calling go from R
author: Romain François
date: '2017-05-14'
slug: calling-go-from-r
categories: []
tags:
  - R
  - go
---

Before we go any further, I should disclaim that I know almost nothing about `go`,
although to put this in perspective I did not know much about `C++` before
I started to work on `Rcpp`.

Before I started to cook this recipe, I needed to check some ingredients:
 - `go` can make shared libraries (`.so`)
 - `R` can link to those

Casual sunday browsing led me to tick the first box, with [Building shared libraries in Go: Part 1
](https://www.darkcoding.net/software/building-shared-libraries-in-go-part-1/) which is about
calling `go` from `python`.

The second box is automatically checked through some experience.

```{go, eval = FALSE}
package main

import "C"

//export DoubleIt
func DoubleIt(x int) int {
        return x*2 ;
}

func main() {}
```

The code above (in file `doubler/main.go`) defines the go function `DoubleIt`. By importing the
`"C"` package and exporting the `DoubleIt` symbol we make sure that we we will build
a shared library we can call from `C`.

```
go build -o libdoubler.so -buildmode=c-shared ./doubler
```

We also need some `C` code that will act as a proxy between this shared library and the
C R api can understand. I'll skip over the details, if you have not seen this, it means that
`Rcpp` has done a good job over the years.

```
#include <R.h>
#include <Rinternals.h>

extern int DoubleIt() ;

SEXP godouble(SEXP x){
  return Rf_ScalarInteger( DoubleIt( INTEGER(x)[0] ) ) ;
}
```

So we define a `SEXP`roof function `godouble` that calls `DoubleIt`. We can `R CMD SHLIB` it:

```
$ R CMD SHLIB -L. -ldoubler rgo.c
clang -I/Library/Frameworks/R.framework/Resources/include -DNDEBUG   -I/usr/local/include   -fPIC  -Wall -g -O2  -c rgo.c -o rgo.o
clang -dynamiclib -Wl,-headerpad_max_install_names -undefined dynamic_lookup -single_module -multiply_defined suppress -L/Library/Frameworks/R.framework/Resources/lib -L/usr/local/lib -o rgo.so rgo.o -L. -ldoubler -F/Library/Frameworks/R.framework/.. -framework R -Wl,-framework -Wl,CoreFoundation
```

And then we can finally call it from `R`:

```
$ Rscript -e 'dyn.load("rgo.so"); .Call("godouble", 21L)'
[1] 42
```

