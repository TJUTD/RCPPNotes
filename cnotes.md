VAR(1)
**x**<sub>*t*</sub> = *A***x**<sub>*t* − 1</sub> + **u**<sub>*t*</sub>

``` r
> # coefficent matrix and error terms
> a <- matrix(c(0.5,0.1,0.1,0.5), nrow = 2)
> u <- matrix(rnorm(10000), ncol = 2)
```

``` r
> rSim <- function(coeff, errors) {
+   simdata <- matrix(0, nrow(errors), ncol(errors))
+   for (row in 2:nrow(errors)) {
+     simdata[row,] = coeff %*% simdata[(row-1),] + errors[row,]
+   }
+   return(simdata)
+ }
> 
> rData <- rSim(a, u) 
> par(mfrow = c(1,3))
> matplot(rData, type = 'l', ylab = 'x', xlim = c(4900,5000))
> acf(rData[,1], main = expression(x[1]))
> acf(rData[,2], main = expression(x[2]))
```

![](cnotes_files/figure-markdown_github/VAR_r-1.png)

``` r
> library(inline)
> code <- '
+   arma::mat coeff = Rcpp::as<arma::mat>(a);
+   arma::mat errors = Rcpp::as<arma::mat>(u);
+   int m = errors.n_rows;
+   int n = errors.n_cols;
+   arma::mat simdata(m,n);
+   simdata.row(0) = arma::zeros<arma::mat>(1,n);
+   for (int row = 1; row < m; row++) {
+     simdata.row(row) = simdata.row(row-1)*trans(coeff) + errors.row(row);
+   }
+   return Rcpp::wrap(simdata);
+ '
> 
> # create the compiled function
> rcppSim <- cxxfunction(signature(a = "numeric", u = "numeric"),
+                        body = code,
+                        plugin = "RcppArmadillo")
> rcppData <- rcppSim(a,u) # generated by C++ code
> stopifnot(all.equal(rData, rcppData)) # checking result
```

``` r
> library(compiler)
> compRsim <- cmpfun(rSim)
> library(rbenchmark)
> benchmark(rcppSim(a,u),
+           rSim(a,u),
+           compRsim(a,u),
+           columns = c("test", "replications", "elapsed",
+                       "relative", "user.self", "sys.self"),
+           order="relative")
```

                test replications elapsed relative user.self sys.self
    1  rcppSim(a, u)          100    0.02      1.0      0.02        0
    3 compRsim(a, u)          100    0.93     46.5      0.92        0
    2     rSim(a, u)          100    0.94     47.0      0.94        0

``` r
> src <- '
+   Rcpp::NumericVector xa(a);
+   Rcpp::NumericVector xb(b);
+   int n_xa = xa.size(), n_xb = xb.size();
+ 
+   Rcpp::NumericVector xab(n_xa + n_xb - 1);
+   for (int i = 0; i < n_xa; i++)
+     for (int j = 0; j < n_xb; j++)
+       xab[i + j] += xa[i] * xb[j];
+   return xab;
+ '
> convolution_fun <- cxxfunction(signature(a = "numeric", b = "numeric"), 
+                                body = src, plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c9074176415( SEXP a, SEXP b) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c9074176415(SEXP a, SEXP b) {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::NumericVector xa(a);
      30 :   Rcpp::NumericVector xb(b);
      31 :   int n_xa = xa.size(), n_xb = xb.size();
      32 : 
      33 :   Rcpp::NumericVector xab(n_xa + n_xb - 1);
      34 :   for (int i = 0; i < n_xa; i++)
      35 :     for (int j = 0; j < n_xb; j++)
      36 :       xab[i + j] += xa[i] * xb[j];
      37 :   return xab;
      38 : 
      39 : END_RCPP
      40 : }

``` r
> convolution_fun( 1:4, 2:5 )
```

    [1]  2  7 16 30 34 31 20

includes

``` r
> inc <- '
+   template <typename T>
+   class square : public std::unary_function<T,T> {
+   public:
+     T operator()( T t) const { return t*t ;}
+   };
+ '
> src <- '
+   double x = Rcpp::as<double>(xs);
+   int i = Rcpp::as<int>(is);
+   square<double> sqdbl;
+   square<int> sqint;
+   Rcpp::DataFrame df =
+     Rcpp::DataFrame::create(Rcpp::Named("x", sqdbl(x)),
+     Rcpp::Named("i", sqint(i)));
+   return df;
+ '
> fun <- cxxfunction(signature(xs = "numeric",
+                              is = "integer"),
+                    body = src, include = inc, 
+                    plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 :   template <typename T>
      20 :   class square : public std::unary_function<T,T> {
      21 :   public:
      22 :     T operator()( T t) const { return t*t ;}
      23 :   };
      24 : 
      25 : 
      26 : // declarations
      27 : extern "C" {
      28 : SEXP file1c90130677c( SEXP xs, SEXP is) ;
      29 : }
      30 : 
      31 : // definition
      32 : SEXP file1c90130677c(SEXP xs, SEXP is) {
      33 : BEGIN_RCPP
      34 : 
      35 :   double x = Rcpp::as<double>(xs);
      36 :   int i = Rcpp::as<int>(is);
      37 :   square<double> sqdbl;
      38 :   square<int> sqint;
      39 :   Rcpp::DataFrame df =
      40 :     Rcpp::DataFrame::create(Rcpp::Named("x", sqdbl(x)),
      41 :     Rcpp::Named("i", sqint(i)));
      42 :   return df;
      43 : 
      44 : END_RCPP
      45 : }

``` r
> fun(2.2, 3L)
```

         x i
    1 4.84 9

plugin

``` r
> src <- '
+   Rcpp::NumericVector yr(ys);
+   Rcpp::NumericMatrix Xr(Xs);
+   int n = Xr.nrow(), k = Xr.ncol();
+ 
+   arma::mat X(Xr.begin(), n, k, false);
+   arma::colvec y(yr.begin(), yr.size(), false);
+ 
+   arma::colvec coef = arma::solve(X, y); // fit y ~ X
+   arma::colvec res = y - X*coef; // residuals
+ 
+   double s2 = std::inner_product(res.begin(),res.end(),
+                                  res.begin(),double())
+                                  / (n - k);
+   arma::colvec se = arma::sqrt(s2 *
+                     arma::diagvec(arma::inv(arma::trans(X)*X)));
+ 
+   return Rcpp::List::create(Rcpp::Named("coef")= coef,
+                             Rcpp::Named("se") = se,
+                             Rcpp::Named("df") = n-k);
+ '
> fun <- cxxfunction(signature(ys="numeric",
+                              Xs="numeric"),
+                    body = src, 
+                    plugin="RcppArmadillo", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = $(SHLIB_OPENMP_CFLAGS) $(LAPACK_LIBS) $(BLAS_LIBS) $(FLIBS)
    PKG_CPPFLAGS = -I../inst/include $(SHLIB_OPENMP_CFLAGS)

     >> LinkingTo : RcppArmadillo, Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/RcppArmadillo/include" -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : #include <RcppArmadillo.h>
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c905bfe66e2( SEXP ys, SEXP Xs) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c905bfe66e2(SEXP ys, SEXP Xs) {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::NumericVector yr(ys);
      30 :   Rcpp::NumericMatrix Xr(Xs);
      31 :   int n = Xr.nrow(), k = Xr.ncol();
      32 : 
      33 :   arma::mat X(Xr.begin(), n, k, false);
      34 :   arma::colvec y(yr.begin(), yr.size(), false);
      35 : 
      36 :   arma::colvec coef = arma::solve(X, y); // fit y ~ X
      37 :   arma::colvec res = y - X*coef; // residuals
      38 : 
      39 :   double s2 = std::inner_product(res.begin(),res.end(),
      40 :                                  res.begin(),double())
      41 :                                  / (n - k);
      42 :   arma::colvec se = arma::sqrt(s2 *
      43 :                     arma::diagvec(arma::inv(arma::trans(X)*X)));
      44 : 
      45 :   return Rcpp::List::create(Rcpp::Named("coef")= coef,
      46 :                             Rcpp::Named("se") = se,
      47 :                             Rcpp::Named("df") = n-k);
      48 : 
      49 : END_RCPP
      50 : }

``` r
> x <- rnorm(30)
> y <- 2 + 5*x + rnorm(30)
> x <- cbind(rep(1,30),x)
> fun(y, x)
```

    $coef
             [,1]
    [1,] 2.133406
    [2,] 5.034368

    $se
              [,1]
    [1,] 0.1890157
    [2,] 0.2108935

    $df
    [1] 28

exception handling

``` r
> src <- '
+   int dx = Rcpp::as<int>(x);
+   if( dx > 10 )
+     throw std::range_error("too big");  
+   return Rcpp::wrap( dx * dx);
+ '
> fun <- cxxfunction(signature(x = "integer"), 
+                    body = src, 
+                    plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c901cb842b3( SEXP x) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c901cb842b3(SEXP x) {
      27 : BEGIN_RCPP
      28 : 
      29 :   int dx = Rcpp::as<int>(x);
      30 :   if( dx > 10 )
      31 :     throw std::range_error("too big");  
      32 :   return Rcpp::wrap( dx * dx);
      33 : 
      34 : END_RCPP
      35 : }

``` r
> fun(3)
```

    [1] 9

``` r
> # fun(30)
> # Error in fun(30) : too big
> # fun('abc')
> # Error in fun("abc") : Not compatible with requested type: [type=character; target=integer].
```

IntegerVector class

``` r
> src <- '
+   Rcpp::IntegerVector vec(vx);
+   int prod = 1;
+   for (int i=0; i<vec.size(); i++) {
+     prod *= vec[i];
+   }
+   return Rcpp::wrap(prod);
+ '
> fun <- cxxfunction(signature(vx = "integer"), 
+                    body = src,
+                    plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c901a067504( SEXP vx) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c901a067504(SEXP vx) {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::IntegerVector vec(vx);
      30 :   int prod = 1;
      31 :   for (int i=0; i<vec.size(); i++) {
      32 :     prod *= vec[i];
      33 :   }
      34 :   return Rcpp::wrap(prod);
      35 : 
      36 : END_RCPP
      37 : }

``` r
> fun(1:10) 
```

    [1] 3628800

``` r
> src <- '
+   Rcpp::IntegerVector vec(vx);
+   return Rcpp::wrap(std::accumulate(vec.begin(),vec.end(),
+                               1, std::multiplies<int>()));
+ '
> fun <- cxxfunction(signature(vx = "integer"),  
+                    body = src,
+                    plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c9010652b6e( SEXP vx) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c9010652b6e(SEXP vx) {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::IntegerVector vec(vx);
      30 :   return Rcpp::wrap(std::accumulate(vec.begin(),vec.end(),
      31 :                               1, std::multiplies<int>()));
      32 : 
      33 : END_RCPP
      34 : }

``` r
> fun(1:10)
```

    [1] 3628800

NumericVector class

``` r
> src <- '
+   Rcpp::NumericVector invec(vx);
+   Rcpp::NumericVector outvec(vx);
+   for (int i=0; i<invec.size(); i++) {
+     outvec[i] = log(invec[i]);
+   }
+   return outvec;
+ '
> fun <- cxxfunction(signature(vx = "numeric"),
+                    body = src, plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c9058cc331d( SEXP vx) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c9058cc331d(SEXP vx) {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::NumericVector invec(vx);
      30 :   Rcpp::NumericVector outvec(vx);
      31 :   for (int i=0; i<invec.size(); i++) {
      32 :     outvec[i] = log(invec[i]);
      33 :   }
      34 :   return outvec;
      35 : 
      36 : END_RCPP
      37 : }

``` r
> x <- seq(1.0, 3.0, by = 1)
> cbind(x, fun(x))
```

                 x          
    [1,] 0.0000000 0.0000000
    [2,] 0.6931472 0.6931472
    [3,] 1.0986123 1.0986123

    - clone

``` r
> src <- '
+   Rcpp::NumericVector invec(vx);
+   Rcpp::NumericVector outvec = Rcpp::clone(vx);
+   for (int i=0; i<invec.size(); i++) {
+     outvec[i] = log(invec[i]);
+   }
+   return outvec;
+ '
> fun <- cxxfunction(signature(vx = "numeric"),
+                    body = src, plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c9072906b24( SEXP vx) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c9072906b24(SEXP vx) {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::NumericVector invec(vx);
      30 :   Rcpp::NumericVector outvec = Rcpp::clone(vx);
      31 :   for (int i=0; i<invec.size(); i++) {
      32 :     outvec[i] = log(invec[i]);
      33 :   }
      34 :   return outvec;
      35 : 
      36 : END_RCPP
      37 : }

``` r
> x <- seq(1.0, 3.0, by = 1)
> cbind(x, fun(x))
```

         x          
    [1,] 1 0.0000000
    [2,] 2 0.6931472
    [3,] 3 1.0986123

    - sugar

``` r
> src <- '
+   Rcpp::NumericVector invec(vx);
+   Rcpp::NumericVector outvec = log(invec);
+   return outvec;
+ '
> fun <- cxxfunction(signature(vx = "numeric"),
+                    body = src, plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c9064853113( SEXP vx) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c9064853113(SEXP vx) {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::NumericVector invec(vx);
      30 :   Rcpp::NumericVector outvec = log(invec);
      31 :   return outvec;
      32 : 
      33 : END_RCPP
      34 : }

``` r
> x <- seq(1.0, 3.0, by = 1)
> cbind(x, fun(x))
```

         x          
    [1,] 1 0.0000000
    [2,] 2 0.6931472
    [3,] 3 1.0986123

NumericMatrix

``` r
> src <- '
+   Rcpp::NumericMatrix mat = Rcpp::clone<Rcpp::NumericMatrix>(mx);
+   std::transform(mat.begin(), mat.end(), mat.begin(), ::sqrt);
+   return mat;
+ '
> fun <- cxxfunction(signature(mx = "numeric"),
+                    body = src, plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c9039667bcc( SEXP mx) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c9039667bcc(SEXP mx) {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::NumericMatrix mat = Rcpp::clone<Rcpp::NumericMatrix>(mx);
      30 :   std::transform(mat.begin(), mat.end(), mat.begin(), ::sqrt);
      31 :   return mat;
      32 : 
      33 : END_RCPP
      34 : }

``` r
> orig <- matrix(1:9, 3, 3)
> fun(orig)
```

             [,1]     [,2]     [,3]
    [1,] 1.000000 2.000000 2.645751
    [2,] 1.414214 2.236068 2.828427
    [3,] 1.732051 2.449490 3.000000

Named class

``` r
> src <- '
+   Rcpp::NumericVector x = 
+       Rcpp::NumericVector::create(
+           Rcpp::Named("mean") = 1.23,
+           Rcpp::Named("dim") = 42,
+           Rcpp::Named("cnt") = 12);
+   return x; 
+ '
> fun <- cxxfunction(signature(), body = src, plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c901b761892( ) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c901b761892() {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::NumericVector x = 
      30 :       Rcpp::NumericVector::create(
      31 :           Rcpp::Named("mean") = 1.23,
      32 :           Rcpp::Named("dim") = 42,
      33 :           Rcpp::Named("cnt") = 12);
      34 :   return x; 
      35 : 
      36 : END_RCPP
      37 : }

``` r
> fun()
```

     mean   dim   cnt 
     1.23 42.00 12.00 

``` r
> src <- '
+   Rcpp::NumericVector x = 
+       Rcpp::NumericVector::create(
+           _["mean"] = 1.23,
+           _["dim"] = 42,
+           _["cnt"] = 12);
+   return x; 
+ '
> fun <- cxxfunction(signature(), body = src, plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c9053683e63( ) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c9053683e63() {
      27 : BEGIN_RCPP
      28 : 
      29 :   Rcpp::NumericVector x = 
      30 :       Rcpp::NumericVector::create(
      31 :           _["mean"] = 1.23,
      32 :           _["dim"] = 42,
      33 :           _["cnt"] = 12);
      34 :   return x; 
      35 : 
      36 : END_RCPP
      37 : }

``` r
> fun()
```

     mean   dim   cnt 
     1.23 42.00 12.00 

DataFrame class

``` r
> src <- '
+     Rcpp::IntegerVector v = Rcpp::IntegerVector::create(7,8,9);
+     std::vector<std::string> s(3);
+     s[0] = "x";
+     s[1] = "y";
+     s[2] = "z";
+     return Rcpp::DataFrame::create(Rcpp::Named("a")=v,
+     Rcpp::Named("b")=s);
+ '
> fun <- cxxfunction(signature(), body = src, plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c902e7c6b88( ) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c902e7c6b88() {
      27 : BEGIN_RCPP
      28 : 
      29 :     Rcpp::IntegerVector v = Rcpp::IntegerVector::create(7,8,9);
      30 :     std::vector<std::string> s(3);
      31 :     s[0] = "x";
      32 :     s[1] = "y";
      33 :     s[2] = "z";
      34 :     return Rcpp::DataFrame::create(Rcpp::Named("a")=v,
      35 :     Rcpp::Named("b")=s);
      36 : 
      37 : END_RCPP
      38 : }

``` r
> fun()
```

      a b
    1 7 x
    2 8 y
    3 9 z

R mathematics library

``` r
> src <- '
+     Rcpp::NumericVector x(xx);
+     int n = x.size();
+     Rcpp::NumericVector y1(n),y2(n),y3(n);
+   
+     for (int i=0; i<n; i++) {
+         // accessing function via remapped R header
+         y1[i] = ::Rf_pnorm5(x[i], 0.0, 1.0, 1, 0);
+ 
+         // or accessing same function via Rcpps namespace R
+         y2[i] = R::pnorm(x[i], 0.0, 1.0, 1, 0);
+     }
+     // or using Rcpp sugar which is vectorized
+     y3 = Rcpp::pnorm(x);
+ 
+     return Rcpp::DataFrame::create( Rcpp::Named("R") = y1,
+                                     Rcpp::Named("Rf_") = y2,
+                                     Rcpp::Named("sugar") = y3);
+ '
> fun <- cxxfunction(signature(xx = "numeric"), body = src, plugin = "Rcpp", verbose = T)
```

     >> setting environment variables: 
    PKG_LIBS = 

     >> LinkingTo : Rcpp
    CLINK_CPPFLAGS =  -I"C:/Users/Windows/AppData/Local/R/win-library/4.4/Rcpp/include" 

     >> Program source :

       1 : 
       2 : // includes from the plugin
       3 : 
       4 : #include <Rcpp.h>
       5 : 
       6 : 
       7 : #ifndef BEGIN_RCPP
       8 : #define BEGIN_RCPP
       9 : #endif
      10 : 
      11 : #ifndef END_RCPP
      12 : #define END_RCPP
      13 : #endif
      14 : 
      15 : using namespace Rcpp;
      16 : 
      17 : // user includes
      18 : 
      19 : 
      20 : // declarations
      21 : extern "C" {
      22 : SEXP file1c90fec416a( SEXP xx) ;
      23 : }
      24 : 
      25 : // definition
      26 : SEXP file1c90fec416a(SEXP xx) {
      27 : BEGIN_RCPP
      28 : 
      29 :     Rcpp::NumericVector x(xx);
      30 :     int n = x.size();
      31 :     Rcpp::NumericVector y1(n),y2(n),y3(n);
      32 :   
      33 :     for (int i=0; i<n; i++) {
      34 :         // accessing function via remapped R header
      35 :         y1[i] = ::Rf_pnorm5(x[i], 0.0, 1.0, 1, 0);
      36 : 
      37 :         // or accessing same function via Rcpps namespace R
      38 :         y2[i] = R::pnorm(x[i], 0.0, 1.0, 1, 0);
      39 :     }
      40 :     // or using Rcpp sugar which is vectorized
      41 :     y3 = Rcpp::pnorm(x);
      42 : 
      43 :     return Rcpp::DataFrame::create( Rcpp::Named("R") = y1,
      44 :                                     Rcpp::Named("Rf_") = y2,
      45 :                                     Rcpp::Named("sugar") = y3);
      46 : 
      47 : END_RCPP
      48 : }

``` r
> fun(1:5)
```

              R       Rf_     sugar
    1 0.8413447 0.8413447 0.8413447
    2 0.9772499 0.9772499 0.9772499
    3 0.9986501 0.9986501 0.9986501
    4 0.9999683 0.9999683 0.9999683
    5 0.9999997 0.9999997 0.9999997
