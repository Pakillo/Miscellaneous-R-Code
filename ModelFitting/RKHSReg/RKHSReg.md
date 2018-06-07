# Reproducing Kernel Hilbert Space Regression

# Introduction
This R code comes from Reproducing Kernel Hilbert Spaces for Penalized Regression: A tutorial, [Nosedal-Sanchez et al. (2010)](http://www.tandfonline.com/doi/abs/10.1080/00031305.2012.678196), specifically, their code in the appendix.  The original code had several issues, most of which has been cleaned up. Blocked text represents quotes taken directly from the article, while I add additional notes elsewhere. I leave their inline comments intact for the most part, though in some places the deletion of unnecessary code may put them a bit out of context. I also may add my own in a few empty spots, and made some textual and code corrections.  The purled code (i.e. with no text) can be found in this same directory/filename with .R extension.

# Ridge Regression

>
We consider a sample of size n = 20, ($y_1, y_2, y_3, ..., y_{20}$), from the model
$$ y_i = β_0 + β_1x1_i + β_2x2_i + ϵ_i $$
where $β_0 = 2$, $β_1 = 3$, $β_2 = 0$ and $ϵ_i$ has a N(0, 0.2^2^) . The distribution of the covariates, x1 and
x2, is uniform in [0, 1]^2^ so that x2 is uninformative. The following lines of code generate the data.

## Data

```r
set.seed(3)
n = 20
x1 = runif(n)
x2 = runif(n)
X = matrix(c(x1, x2), ncol = 2) # design matrix
y = 2 + 3 * x1 + rnorm(n, sd = 0.25)
```

## Functions
### Function to find the inverse of a matrix

MC note: can use base::solve, though this function avoids a computationally singular result.


```r
my.inv = function(X, eps = 1e-12) {
  eig.X = eigen(X, symmetric = T)
  P = eig.X[[2]] 
  lambda = eig.X[[1]] 
  ind = lambda > eps
  lambda[ind] = 1/lambda[ind] 
  lambda[!ind] = 0
  ans = P %*% diag(lambda) %*% t(P)
  return(ans)
}
```

### Reproducing Kernel

```r
rk = function(s, t) {
  crossprod(s, t)              # MC note: slightly more succinct than the original double loop; also sum(s*t)
} 
```


### Gram matrix
MC note: for the first example involving ridge regression, the get.gramm function just produces tcrossprod(X).  I generalize it in case a different kernel is desired and add that as an additional argument.  This will avoid having to redo the function later as the authors did in their appendix.


```r
get.gramm = function(X, rkfunc=rk) {
  apply(X, 1, function(Row) apply(X, 1, function(tRow) rkfunc(Row, tRow)))  
}
```


### Ridge regression


```r
ridge.regression = function(X, y, lambda) {
  Gramm = get.gramm(X)                               # Gramm matrix (nxn)
  n = length(y)                                      # n=length of y
  Q = cbind(1, Gramm)                                # design matrix
  S = rbind(0, cbind(0, Gramm))
  M = crossprod(Q) + lambda*S
  M.inv = my.inv(M)                                  # inverse of M
  gamma.hat = crossprod(M.inv, crossprod(Q, y))
  f.hat = Q %*% gamma.hat
  A = Q %*% M.inv %*% t(Q)
  tr.A = sum(diag(A))                                # trace of hat matrix
  rss = crossprod(y - f.hat)                         # residual sum of squares
  gcv = n*rss / (n - tr.A)^2                         # obtain GCV score
  return(list(f.hat = f.hat, gamma.hat = gamma.hat, gcv = gcv))
}
```

## Model results

> A simple direct search for the GCV optimal smoothing parameter can be made as follows:




```r
lambda = 1e-8
reps = 40
lambda = 1.5^(1:reps-1) * lambda
V = sapply(lambda, function(lam) ridge.regression(X, y, lam)$gcv)
```

<img src="RKHSReg_files/figure-html/unnamed-chunk-7-1.png" title="" alt="" style="display: block; margin: auto;" />



> The GCV plot produced by this code is displayed in Figure A.2. Now, by following Section 4.2, $\hat{β_0}$, $\hat{β_1}$ and $\hat{β_2}$ can be obtained as follows.


```r
opt.mod = ridge.regression(X, y, lambda[which.min(V)])     # fit optimal model

### finding beta.0, beta.1 and beta.2 ##########
gamma.hat = opt.mod$gamma.hat
beta.hat.0 = opt.mod$gamma.hat[1]               # intercept
beta.hat = crossprod(gamma.hat[-1,], X)         # slope and noise term coefficients
```

> The resulting estimates are: $\hat{β_0}$ = 2.1253, $\hat{β_1}$ = 2.6566 and $\hat{β_2}$ = 0.1597.

MC note: I add a comparison to glmnet, where alpha=0 is equivalent to ridge regression.


```r
c(beta.hat.0, beta.hat)
```

```
## [1] 2.1253369 2.6566261 0.1597225
```

```r
coef(glmnet::glmnet(X, y, alpha=0, lambda=lambda[which.min(V)]))
```

```
## 3 x 1 sparse Matrix of class "dgCMatrix"
##                    s0
## (Intercept) 2.1360199
## V1          2.6394944
## V2          0.1552121
```


# Cubic Spline

> A.2 RKHS solution applied to Cubic Smoothing Spline <br>
We consider a sample of size n = 50, ($y_1, y_2, y_3, ..., y_{50}$), from the model 
$y_i = sin(2πx_i) + ϵ_i$ where ϵ~i~ has a N(0, 0.2^2^) . The following code generates x
and y...

## Data 

```r
set.seed(3)
n = 50
x = matrix(runif(n), nrow=n, ncol=1)
x.star = as.matrix(sort(x))                      # sorted x, used by plot
y = sin(2*pi*x.star) + rnorm(n, sd=0.2)
```


> Below we give a function to find the cubic smoothing spline using the RKHS
framework we discussed in Section 4.3. We also provide a graph with our
estimation along with the true function and data.

## Functions

### Reproducing Kernel


```r
rk.1 = function(s, t) {
  return(.5*min(s, t)^2 * max(s, t) - (1/6)*min(s, t)^3)        # sign error fixed here
}
```


MC note: no need to redo the gram function due to previous change that accepts the kernel as an argument

### Smoothing Spline

```r
smoothing.spline = function(X, y, lambda) {
  Gramm = get.gramm(X, rkfunc=rk.1)                  # Gramm matrix (nxn)
  n = length(y)
  J = cbind(1, X)                                    # matrix with a basis for the null space of the penalty; MC note:, never name anything T (True) or t (transpose)!
  Q = cbind(J, Gramm)                                # design matrix
  m = ncol(J)                                        # dimension of the null space of the penalty
  S = matrix(0, n + m, n + m)                        # initialize S
  S[(m + 1):(n + m),(m + 1):(n + m)] = Gramm         # non-zero part of S
  M = crossprod(Q) + lambda*S
  M.inv = my.inv(M)                                  # inverse of M
  gamma.hat = crossprod(M.inv, crossprod(Q, y))
  f.hat = Q %*% gamma.hat
  A = Q %*% M.inv %*% t(Q)
  tr.A = sum(diag(A))                                # trace of hat matrix
  rss = crossprod(y - f.hat)                         # residual sum of squares
  gcv = n * rss/(n - tr.A)^2                         # obtain GCV score
  return(list(f.hat = f.hat, gamma.hat = gamma.hat,gcv = gcv))
}
```


## Model results
> A simple direct search for the GCV optimal smoothing parameter can be made as follows:
Now we have to find an optimal lambda using GCV...


```r
lambda = 1e-8
reps = 60

lambda = 1.5^(1:reps-1) * lambda
V = sapply(lambda, function(lam) smoothing.spline(x.star, y, lam)$gcv)

# Plot of GCV
plot(1:reps, V, type = 'l', main = 'GCV score', xlab = 'i', ylab = 'GCV', bty='n') 
```

<img src="RKHSReg_files/figure-html/unnamed-chunk-13-1.png" title="" alt="" style="display: block; margin: auto;" />

MC note: I've added comparisons to an additive and gaussian process model.


```r
opt.mod.2 = smoothing.spline(x.star, y, lambda[which.min(V)])       # fit optimal model

# comparison models
## mgcv::gam(y ~ s(x.star))
## kernlab::gausspr(y ~ x.star, kernel='splinedot')
```

```
##  Setting default kernel parameters
```

<img src="RKHSReg_files/figure-html/unnamed-chunk-14-1.png" title="" alt="" style="display: block; margin: auto;" />

> This graph is plotted in Figure A.2.








# \*\* ORIGINAL CODE  \*\*

The website where the article is located was no longer displaying supplemental files.  This comes from an earlier draft at http://www.math.unm.edu/~alvaro/rkhs_tutorial_12-06-2010.pdf, so might not be exactly as it was uploaded.  I used Rstudio's default cleanup to make the code a little easier to read, and maybe added a little spacing, but otherwise it is identical to what's in the linked paper.

## A.1

```r
###### Data ########
set.seed(3)
n <- 20
x1 <- runif(n)
x2 <- runif(n)
X <- matrix(c(x1, x2), ncol = 2) # design matrix
y <- 2 + 3 * x1 + rnorm(n, sd = 0.25)

##### function to find the inverse of a matrix ####
my.inv <- function(X, eps = 1e-12) {
  eig.X <- eigen(X, symmetric = T)
  P <- eig.X[[2]] lambda <- eig.X[[1]] ind <- lambda > eps
  lambda[ind] <- 1 / lambda[ind] lambda[!ind] <- 0
  ans <- P %*% diag(lambda, nrow = length(lambda)) %*% t(P)
  return(ans)
}

###### Reproducing Kernel #########
rk <- function(s, t) {
  p <- length(s)
  rk <- 0
  for (i in 1:p) {
    rk <- s[i] * t[i] + rk
  }
  return((rk))
}

##### Gram matrix #######
get.gramm <- function(X) {
  n <- dim(X)[1]
  Gramm <-
    matrix(0, n, n) #initializes Gramm array #i=index for rows
  #j=index for columns Gramm<-as.matrix(Gramm) # Gramm matrix
  for (i in 1:n) {
    for (j in 1:n) {
      Gramm[i, j] <- rk(X[i,], X[j,])
    }
  } return(Gramm)
}


ridge.regression <- function(X, y, lambda) {
  Gramm <- get.gramm(X) #Gramm matrix (nxn)
  n <- dim(X)[1] # n=length of y
  J <- matrix(1, n, 1) # vector of ones dim
  Q <- cbind(J, Gramm) # design matrix
  m <- 1 # dimension of the null space of the penalty
  S <- matrix(0, n + m, n + m) #initialize S
  S[(m + 1):(n + m), (m + 1):(n + m)] <- Gramm #non-zero part of S
  M <- (t(Q) %*% Q + lambda * S)
  M.inv <- my.inv(M) # inverse of M
  gamma.hat <- crossprod(M.inv, crossprod(Q, y))
  f.hat <- Q %*% gamma.hat
  A <- Q %*% M.inv %*% t(Q)
  tr.A <- sum(diag(A)) #trace of hat matrix
  rss <- t(y - f.hat) %*% (y - f.hat) #residual sum of squares
  gcv <- n * rss / (n - tr.A) ^ 2 #obtain GCV score
  return(list(
    f.hat = f.hat,
    gamma.hat = gamma.hat,
    gcv = gcv
  ))
}


# Plot of GCV

lambda <- 1e-8 V <- rep(0, 40) for (i in 1:40) {
  V[i] <- ridge.regression(X, y, lambda)$gcv #obtain GCV score
  lambda <- lambda * 1.5 #increase lambda
}

index <- (1:40) plot(
  1.5 ^ (index - 1) * 1e-8,
  V,
  type = "l",
  main = "GCV
  score",
  lwd = 2,
  xlab = "lambda",
  ylab = "GCV"
) # plot score


i <- (1:60)[V == min(V)] # extract index of min(V)
opt.mod <- ridge.regression(X, y, 1.5 ^ (i - 1) * 1e-8) #fit optimal model

### finding beta.0, beta.1 and beta.2 ##########
gamma.hat <- opt.mod$gamma.hat
beta.hat.0 <- opt.mod$gamma.hat[1]#intercept
beta.hat <- gamma.hat[2:21,] %*% X #slope and noise term coefficients
```

## A.2

```r
###### Data ######## 
set.seed(3) 
n <- 50
x <- matrix(runif(n), nrow, ncol = 1)
x.star <- matrix(sort(x), nrow, ncol = 1) # sorted x, used by plot
y <- sin(2 * pi * x.star) + rnorm(n, sd = 0.2)


#### Reproducing Kernel for <f,g>=int_0^1 f’’(x)g’’(x)dx #####
rk.1 <- function(s, t) {
  return((1 / 2) * min(s, t) ^ 2) * (max(s, t) + (1 / 6) * (min(s, t)) ^ 3)
}

get.gramm.1 <- function(X) {
  n <- dim(X)[1]
  Gramm <- matrix(0, n, n) #initializes Gramm array
  #i=index for rows
  #j=index for columns
  Gramm <- as.matrix(Gramm) # Gramm matrix
  for (i in 1:n) {
    for (j in 1:n) {
      Gramm[i, j] <- rk.1(X[i, ], X[j, ])
    }
  }
  return(Gramm)
}

smoothing.spline <- function(X, y, lambda) {
  Gramm <- get.gramm.1(X) #Gramm matrix (nxn)
  n <- dim(X)[1] # n=length of y
  J <- matrix(1, n, 1) # vector of ones dim
  T <- cbind(J, X) # matrix with a basis for the null space of the penalty
  Q <- cbind(T, Gramm) # design matrix
  m <- dim(T)[2] # dimension of the null space of the penalty
  S <- matrix(0, n + m, n + m) #initialize S
  S[(m + 1):(n + m), (m + 1):(n + m)] <- Gramm #non-zero part of S
  M <- (t(Q) %*% Q + lambda * S)
  M.inv <- my.inv(M) # inverse of M
  gamma.hat <- crossprod(M.inv, crossprod(Q, y))
  f.hat <- Q %*% gamma.hat
  A <- Q %*% M.inv %*% t(Q)
  tr.A <- sum(diag(A)) #trace of hat matrix
  rss <- t(y - f.hat) %*% (y - f.hat) #residual sum of squares
  gcv <- n * rss / (n - tr.A) ^ 2 #obtain GCV score
  return(list(
    f.hat = f.hat,
    gamma.hat = gamma.hat,
    gcv = gcv
  ))
}



### Now we have to find an optimal lambda using GCV...
### Plot of GCV
lambda <- 1e-8 V <- rep(0, 60) for (i in 1:60) {
  V[i] <- smoothing.spline(x.star, y, lambda)$gcv #obtain GCV score
  lambda <- lambda * 1.5 #increase lambda
} 
plot(1:60,
     V,
     type = "l",
     main = "GCV score",
     xlab = "i") # plot score
i <- (1:60)[V == min(V)] # extract index of min(V)
opt.mod.2 <- smoothing.spline(x.star, y, 1.5 ^ (i - 1) * 1e-8) #fit optimal
model

#Graph (Cubic Spline)
plot(
  x.star,
  opt.mod.2$f.hat,
  type = "l",
  lty = 2,
  lwd = 2,
  col = "blue",
  xlab = "x",
  ylim = c(-2.5, 1.5),
  xlim = c(-0.1, 1.1),
  ylab = "response",
  main = "Cubic Spline"
)
#predictions
lines(x.star, sin(2 * pi * x.star), lty = 1, lwd = 2) #true
legend(
  -0.1,
  -1.5,
  c("predictions", "true"),
  lty = c(2, 1),
  bty = "n",
  lwd = c(2, 2),
  col = c("blue", "black")
) 
points(x.star, y)
```