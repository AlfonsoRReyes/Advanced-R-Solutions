
# OO field guide



## S3

1.  __<span style="color:red">Q</span>__: Read the source code for `t()` and `t.test()` and confirm that 
    `t.test()` is an S3 generic and not an S3 method. What happens if 
    you create an object with class `test` and call `t()` with it?  
    __<span style="color:green">A</span>__: We can see that `t.test()` is a generic, because it calls `UseMethod()`
    
    
    ```r
    t.test
    #> function (x, ...) 
    #> UseMethod("t.test")
    #> <bytecode: 0x00000000178332c0>
    #> <environment: namespace:stats>
    ```
    
    If we create an object with class "test" `t.test()` will cause R to call `t.test.default()` unless you create a method `t.test()` for the generic `t()`.

2.  __<span style="color:red">Q</span>__: What classes have a method for the `Math` group generic in base R? Read 
    the source code. How do the methods work?  
    __<span style="color:green">A</span>__: 
    
    
    ```r
    methods("Math")
    #> [1] Math,nonStructure-method Math,structure-method   
    #> [3] Math.data.frame          Math.Date               
    #> [5] Math.difftime            Math.factor             
    #> [7] Math.POSIXt             
    #> see '?methods' for accessing help and source code
    ```

3.  __<span style="color:red">Q</span>__: R has two classes for representing date time data, `POSIXct` and 
    `POSIXlt`, which both inherit from `POSIXt`. Which generics have 
    different behaviours for the two classes? Which generics share the same
    behaviour?  
    __<span style="color:green">A</span>__: Since both inherit from "POSIXt", these should be the same for both classes:  
    
    
    ```r
    methods(class = "POSIXt")
    #>  [1] -            +            all.equal    as.character Axis        
    #>  [6] coerce       cut          diff         hist         initialize  
    #> [11] is.numeric   julian       Math         months       Ops         
    #> [16] pretty       quantile     quarters     round        seq         
    #> [21] show         slotsFromS3  str          trunc        weekdays    
    #> see '?methods' for accessing help and source code
    ```
    
    And these should be different (or only existing for one of the classes):
    
    
    ```r
    methods(class = "POSIXct")
    #>  [1] [             [[            [<-           as.data.frame as.Date      
    #>  [6] as.list       as.POSIXlt    c             coerce        format       
    #> [11] initialize    mean          print         rep           show         
    #> [16] slotsFromS3   split         summary       Summary       weighted.mean
    #> [21] xtfrm        
    #> see '?methods' for accessing help and source code
    methods(class = "POSIXlt")
    #>  [1] [             [<-           anyNA         as.data.frame as.Date      
    #>  [6] as.double     as.matrix     as.POSIXct    c             coerce       
    #> [11] duplicated    format        initialize    is.na         length       
    #> [16] mean          names         names<-       print         rep          
    #> [21] show          slotsFromS3   sort          summary       Summary      
    #> [26] unique        weighted.mean xtfrm        
    #> see '?methods' for accessing help and source code
    ```

4.  __<span style="color:red">Q</span>__: Which base generic has the greatest number of defined methods?  
__<span style="color:green">A</span>__: 
    
    
    ```r
    library(methods)
    objs <- mget(ls("package:base"), inherits = TRUE)
    funs <- Filter(is.function, objs)
    generics <- Filter(function(x) ("generic" %in% pryr::ftype(x)), funs)
    sort(sapply(names(generics),
                function(x) length(methods(x))),
         decreasing = TRUE)[1]
    #> print 
    #>   203
    ```
    
5.  __<span style="color:red">Q</span>__: `UseMethod()` calls methods in a special way. Predict what the following
     code will return, then run it and read the help for `UseMethod()` to 
    figure out what's going on. Write down the rules in the simplest form
    possible.

    
    ```r
    y <- 1
    g <- function(x) {
      y <- 2
      UseMethod("g")
    }
    g.numeric <- function(x) y
    g(10)
    
    h <- function(x) {
      x <- 10
      UseMethod("h")
    }
    h.character <- function(x) paste("char", x)
    h.numeric <- function(x) paste("num", x)
    
    h("a")
    ```
    
    __<span style="color:green">A</span>__: `g(10)` will return `2`. Since `2` is of class numeric, `g.numeric()` is called. It has only `x` in its execution environment and R will search for `y` in the enclosing environment, where `y` is defined as `2`. From `?UseMethod`:
    
    > UseMethod creates a new function call with arguments matched as they came in to the generic. Any local variables defined before the call to UseMethod are retained (unlike S). 
    
    So generics look at the class of their first argument (default) for method dispatch.
    Then a call to the particular method is made. Since the methods are created by the generic, R will look in the generics environment (including all objects defined before (!) the `UseMethod` statement) when an object is not found in the environment of the called method.
    
    `h("a")` will return `"char a"`, because `x = "a"` is given as input to the called method, which is of class character and so `h.character` is called and R also doesn't need to look elsewhere for `x`.
    
6.  __<span style="color:red">Q</span>__: Internal generics don't dispatch on the implicit class of base types.
    Carefully read `?"internal generic"` to determine why the length of `f` 
    and `g` is different in the example below. What function helps 
    distinguish between the behaviour of `f` and `g`?

    
    ```r
    f <- function() 1
    g <- function() 2
    class(g) <- "function"
    
    class(f)
    #> [1] "function"
    class(g)
    #> [1] "function"
    
    length.function <- function(x) "function"
    length(f)
    #> [1] 1
    length(g)
    #> [1] "function"
    ```
    
    __<span style="color:green">A</span>__: From `?"internal generic"`:  
    
    > Many R objects have a class attribute, a character vector giving the names of
    the classes from which the object inherits. If the object does not have a class attribute,
    it has an implicit class, "matrix", "array" or the result of mode(x)
    (except that integer vectors have implicit class "integer").
    (Functions oldClass and oldClass<- get and set the attribute, which can also be done     directly.)

    In the first case, the internal generic `length` does not find the `class` of `f` ("function"), so the method `length.function` is not called. This is because `f` doesn't have a class - which is needed for the S3 method dispatch of internal generics (those that are implemented in C, you can check if they are generics with `pryr::ftype`) - only an implicit class. It is very confusing, because `class(f)` returns this (implicit) class.  
You can check if a class is only implicit by using one of the following approaches:  
    *   `is.object(f)` returns `FALSE`  
    *   `oldClass(f)` returns `NULL`  
    *   `attributes(f)` doesn't contain a `$class` field

## S4

1.  __<span style="color:red">Q</span>__: Which S4 generic has the most methods defined for it? Which S4 class 
    has the most methods associated with it?  
    __<span style="color:green">A</span>__: 
    
    **generics:**
    
    We restrict our search to those packages that everyone should have installed:
    
    
    ```r
    search()
    #> [1] ".GlobalEnv"        "package:methods"   "package:stats"    
    #> [4] "package:graphics"  "package:grDevices" "package:utils"    
    #> [7] "package:datasets"  "Autoloads"         "package:base"
    ```
    
    Then we start our search for generics and keep those of otype S4:
    
    
    ```r
    generics <- getGenerics(where = search())
    is_gen_s4 <- vapply(generics@.Data, 
                        function(x) pryr::otype(get(x)) == "S4", logical(1))
    generics <- generics[is_gen_s4]
    ```
    
    Finally we calculate the S4-generic with the most methods:
    
    
    ```r
    sort(sapply(generics, function(x) length(methods(x))), decreasing = TRUE)[1]
    #> coerce 
    #>     27
    ```
    
    **classes:**
    
    We collect all S4 classes within a character vector:
    
    
    ```r
    s4classes <- getClasses(where = .GlobalEnv, inherits = TRUE)
    ```
    
    Then we are going to steal the following function from [S4 system development in Bioconductor](http://www.bioconductor.org/help/course-materials/2010/AdvancedR/S4InBioconductor.pdf) that returns all methods to a given class

    
    ```r
    s4Methods <- function(class){
      methods <- showMethods(classes = class, printTo = FALSE) # notice the last setting
      methods <- methods[grep("^Function:", methods)]
      sapply(strsplit(methods, " "), "[", 2)
    }
    ```
    
    Finally we apply this function to get the methods of each class and format a little bit to answer the question:
    
    
    ```r
    s4class_methods <- lapply(s4classes, s4Methods)
    names(s4class_methods) <- s4classes
    sort(lengths(s4class_methods), decreasing = TRUE)[1]
    #> ANY 
    #>  66
    ```

2.  __<span style="color:red">Q</span>__: What happens if you define a new S4 class that doesn't "contain" an 
    existing class?  (Hint: read about virtual classes in `?Classes`.)  
    __<span style="color:green">A</span>__: Since `?Classes` is deprecated we refer to `?setClass`:
    
    > Calls to setClass() will also create a virtual class, either when only the Class argument is supplied (no slots or superclasses) or when the contains= argument includes the special class name "VIRTUAL".
    >
    > In the latter case, a virtual class may include slots to provide some common behavior without fully defining the object—see the class traceable for an example. Note that "VIRTUAL" does not carry over to subclasses; a class that contains a virtual class is not itself automatically virtual.

3.  __<span style="color:red">Q</span>__: What happens if you pass an S4 object to an S3 generic? What happens 
    if you pass an S3 object to an S4 generic? (Hint: read `?setOldClass` 
    for the second case.)  
    __<span style="color:green">A</span>__: 

## RC

1.  __<span style="color:red">Q</span>__: Use a field function to prevent the account balance from being directly
    manipulated. (Hint: create a "hidden" `.balance` field, and read the 
    help for the fields argument in `setRefClass()`.)  
    __<span style="color:green">A</span>__: We are not that experienced in general RC classes, but it is easy with R6 classes. You can find all the information you need [here](https://github.com/wch/R6). To solve the exercise this [introduction](https://cran.r-project.org/web/packages/R6/vignettes/Introduction.html) should be sufficient:
    
    
    ```r
    # definition of the class
    Account2 <- R6::R6Class("Account",
                            public = list(
                              initialize = function(balance = 0){
                                private$balance = balance
                                },
                              withdraw = function(x){
                                if (private$balance < x) stop("Not enough money")
                                private$balance <- private$balance - x
                                },
                              deposit = function(x) {
                                private$balance <- private$balance + x
                                }
                              ),
                            private = list(
                              balance = NULL
                              )
                            )
    # Checking the behaviour
    # a <- Account2$new(100)
    # a$withdraw(50); a
    # a$balance
    # a$balance <- 5000
    # a$deposit(100); a
    # a$withdraw(200); a
    ```

2.  __<span style="color:red">Q</span>__: I claimed that there aren't any RC classes in base R, but that was a 
    bit of a simplification. Use `getClasses()` and find which classes 
    `extend()` from `envRefClass`. What are the classes used for? (Hint: 
    recall how to look up the documentation for a class.)  
    __<span style="color:green">A</span>__: We get these classes as described in the exercise:
    
    
    ```r
    classes <- getClasses(where = .GlobalEnv, inherits = TRUE)
    classes[unlist(lapply(classes, function(x) methods::extends(x, "envRefClass")))]
    #> [1] "envRefClass"      "refGeneratorSlot" "localRefClass"
    ```
    
    Their need is best described in `class?envRefClass` "Purpose of the Class":

    > This class implements basic reference-style semantics for R objects. Objects normally do not come directly from this class, but from subclasses defined by a call to setRefClass. The documentation below is technical background describing the implementation, but applications should use the interface documented under setRefClass, in particular the $ operator and field accessor functions as described there.
