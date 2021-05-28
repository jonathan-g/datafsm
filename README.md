<!-- README.md is generated from README.Rmd. Please edit that file -->
<!-- badges: start -->

[![CRAN_Status_Badge](https://www.r-pkg.org/badges/version-last-release/datafsm)](https://cran.r-project.org/package=datafsm)
[![DOI](https://zenodo.org/badge/94561899.svg)](https://zenodo.org/badge/latestdoi/94561899)
[![License:
MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![R-CMD-check](https://github.com/jonathan-g/datafsm/workflows/R-CMD-check/badge.svg)](https://github.com/jonathan-g/datafsm/actions)
<!-- badges: end -->

This package implements our method for automatically generating models
of dynamic decision-making that both have strong predictive power and
are interpretable in human terms. We use an efficient model
representation and a genetic algorithm-based estimation process to
generate simple deterministic approximations that explain most of the
structure of complex stochastic processes. The genetic algorithm is
implemented with the **GA** package ([Scrucca
2013](https://www.jstatsoft.org/v53/i04/)). Our method, implemented in
C++ and R, scales well to large data sets. We have applied the package
to empirical data, and demonstrated the method’s ability to recover
known data-generating processes by simulating data with agent-based
models and correctly deriving the underlying decision models for
multiple agent models and degrees of stochasticity.

A user of our package can estimate models by providing their data in a
common “panel data” format. The package is designed to estimate time
series classification models that use a small number of binary predictor
variables and move back and forth between the values of the outcome
variable over time. Larger sets of predictor variables can be reduced to
smaller sets with cross-validation. Although the predictor variables
must be binary, a quantitative variable can be converted into binary by
division of the observed values into high/low classes. Future releases
of the package may include additional estimation methods to complement
genetic algorithm optimization.

``` r
# Load and attach datafsm into your R session, making its functions available:
library(datafsm)
```

Please cite the package if you use it with the text generated by:

``` r
citation("datafsm")
#> 
#> To cite datafsm in publications use:
#> 
#>   John J. Nay, Jonathan M. Gilligan (2015) "Data-Driven Dynamic
#>   Decision Models," in L. Yilmaz et al. (eds.), Proc. 2015 Winter
#>   Simulation Conf.
#> 
#> A BibTeX entry for LaTeX users is
#> 
#>   @InProceedings{,
#>     title = {Data-Driven Dynamic Decision Models},
#>     author = {John J. Nay and Jonathan M. Gilligan},
#>     booktitle = {Proceedings of the 2015 Winter Simulation Conference},
#>     year = {2015},
#>     editor = {L. Yilmaz and W.K.V. Chan and I. Moon and T.M.K. Roeder and C. Macal and M. Rosetti},
#>   }
```

# Fake Data Example

To quickly show it works, we can create fake data. Here, we generate
1000 repetitions of a 10-round game in which each player starts by
making a random move, and in subsequent rounds, each player follows a
“noisy tit-for-tat” strategy that’s equivalent to tit-for-tat, except
that with a 5% probability the player will make a random move.

``` r
rounds <- 10
reps <- 1000
noise_level <- 0.05
```

``` r
seed1 <- 1372456261
seed2 <-  597693057
seed3 <-  805078823
set.seed(seed1)
cdata <- data.frame(outcome = NA,
                    period = rep(seq(rounds), reps),
                    my.decision1 = NA,
                    other.decision1 = NA)
#
# Prisoner's dilemma
#
pd_outcome <- function(player_1, player_2) {
  #
  # 1 = C
  # 2 = D
  #
  player_1  + 1
}

tit_for_tat <- function(last_round_self, last_round_opponent) {
    last_round_opponent
}

noisy_tit_for_tat <- function(last_round_self, last_round_opponent, noise_level) {
  if (runif(1,0,1) <= noise_level) {
    sample(0:1,1)
  } else {
    last_round_opponent
  }
}

for (i in seq_along(cdata$period)) {
  if (cdata$period[i] == 1) {
    my.decision <- sample(0:1,1, prob = c(1 - noise_level, noise_level))
    other.decision <- sample(0:1,1, prob = c(1 - noise_level, noise_level))
    cdata[i, "outcome"] <- pd_outcome(my.decision, other.decision)
  } else{
    my.last <- my.decision
    other.last <- other.decision
    my.decision <- noisy_tit_for_tat(my.last, other.last, noise_level)
    other.decision <- noisy_tit_for_tat(other.last, my.last, noise_level)
    cdata[i,c("outcome", "my.decision1", "other.decision1")] <- 
      c(pd_outcome(my.decision, other.decision), my.last, other.last)
  }
}
```

The only required argument of the main function of the package,
`evolve_model`, is a `data.frame` object, which must have 3-5 columns.
The first two columns must be named “period” and “outcome” (period is
the time period that the outcome action was taken). The remaining one to
three columns are predictors, and may have arbitrary names. Each row of
the `data.frame` is an observational unit, an action taken at a
particular time and any relevant variables for that time. All of the
(3-5 columns) should be named. The period and outcome columns should be
integer vectors — e.g. `c(1,2,1)` — and the columns with the predictor
variable data should be logical vectors — e.g. `c(TRUE, FALSE, FALSE)` —
or vectors that can be coerced to logical with `as.logical()`.

Here are the first eleven rows of this fake data:

| outcome | period | my.decision1 | other.decision1 |
|--------:|-------:|-------------:|----------------:|
|       1 |      1 |           NA |              NA |
|       1 |      2 |            0 |               0 |
|       1 |      3 |            0 |               0 |
|       2 |      4 |            0 |               0 |
|       1 |      5 |            1 |               0 |
|       2 |      6 |            0 |               1 |
|       1 |      7 |            1 |               0 |
|       2 |      8 |            0 |               1 |
|       1 |      9 |            1 |               0 |
|       2 |     10 |            0 |               1 |
|       1 |      1 |           NA |              NA |

We can estimate a model with that data. `evolve_model` uses a genetic
algorithm to estimate a finite-state machine (FSM) model, primarily for
understanding and predicting decision-making. This is the main function
of the `datafsm` package. It relies on the `GA` package for genetic
algorithm optimization. We chose to use a GA because GAs perform well in
rugged search spaces to solve integer optimization problems, are a
natural complement to our binary representation of FSMs, and are easily
parallelized.

`evolve_model` takes data on predictors and data on the outcome and
automatically creates a fitness function that takes training data, an
action vector that `evolve_model` generates, and a state matrix
`evolve_model` generates as input and returns numeric vector whose
length is the number of rows in the training data. `evolve_model` then
computes a fitness score for that potential solution FSM by comparing it
to the provided `outcome` in the real training data. This is repeated
for every FSM in the population and then the probability of selection
for the next generation is proportional to the fitness scores. If the
argument `cv` is set to `TRUE` the function will call itself recursively
while varying the number of states inside a cross-validation loop in
order to estimate the optimal number of states, then it will set the
number of states to that optimal number and estimate the model on the
full training set.

If the argument `parallel` is set to `TRUE`, then these evaluations are
distributed across the available processors of the computer using the
`doParallel` functions; otherwise, the evaluations of fitness are
conducted sequentially. Because this fitness function that
`evolve_model` creates must loop through all the training data every
time it is evaluated and we need to evaluate many possible solution
FSMs, the fitness function is implemented in C++ to improve its
performance.

``` r
set.seed(seed2)
res <- evolve_model(cdata, seed = seed3)
```

`evolve_model` evolves the models on training data and then, if a test
set is provided, uses the best solution to make predictions on test
data. Finally, the function returns the GA object and the decoded
version of the best string in the population. Formally, the function
return an `S4` object with slots for:

-   `call` Language from the call of the function.
-   `actions` Numeric vector with the number of actions.
-   `states` Numeric vector with the number of states.
-   `GA` S4 object created by `ga()` from the `GA` package.
-   `state_mat` Numeric matrix with rows as states and columns as
    predictors.
-   `action_vec` Numeric vector indicating what action to take for each
    state.
-   `predictive` Numeric vector of length one with test data accuracy if
    test data was supplied; otherwise, a character vector with a message
    that the user should provide test data for better estimate of
    generalizable performance.
-   `varImp` Numeric vector with length equal to the number of columns
    in the state matrix, containing relative importance scores for each
    predictor.
-   `timing` Numeric vector length one containing the total elapsed time
    it took `evolve_model` to execute.
-   `diagnostics` Character vector length one, designed to be printed
    with `base::cat()`.

Use the summary and plot methods on the output of the `evolve_model()`
function:

``` r
summary(res)
#> Length  Class   Mode 
#>      1 ga_fsm     S4
plot(res, action_label = ifelse(action_vec(res)==1, "C", "D"), 
     transition_label = c('cc','dc','cd','dd'))
```

![Result of `plot()` method call on `ga_fsm`
object.](man/figures/README/plot.fsm-1.png)

The diagram shows that `evolve_model` recovered a tit-for-tat model in
which the player in question (“me”) mimics the last action of the
opponent.

Use the `estimation_details` method on the output of the
`evolve_model()` function:

``` r
suppressMessages(library(GA))
plot(estimation_details(res))
```

![Result of `plot()` method call on ga object, which is obtained by
calling `estimation_details()` on `ga_fsm`
object.](man/figures/README/plot.evolution-1.png)

# Acknowledgements

This work is supported by U.S. National Science Foundation grants
EAR-1416964 and EAR-1204685.

# Session Info

``` r
sessionInfo()
#> R version 4.1.0 (2021-05-18)
#> Platform: x86_64-w64-mingw32/x64 (64-bit)
#> Running under: Windows 10 x64 (build 19043)
#> 
#> Matrix products: default
#> 
#> locale:
#> [1] LC_COLLATE=English_United States.1252 
#> [2] LC_CTYPE=English_United States.1252   
#> [3] LC_MONETARY=English_United States.1252
#> [4] LC_NUMERIC=C                          
#> [5] LC_TIME=English_United States.1252    
#> 
#> attached base packages:
#> [1] stats     graphics  grDevices utils     datasets  methods   base     
#> 
#> other attached packages:
#> [1] GA_3.2.1         iterators_1.0.13 foreach_1.5.1    datafsm_0.2.4   
#> 
#> loaded via a namespace (and not attached):
#>  [1] Rcpp_1.0.6           lubridate_1.7.10     lattice_0.20-44     
#>  [4] class_7.3-19         assertthat_0.2.1     digest_0.6.27       
#>  [7] ipred_0.9-11         utf8_1.2.1           R6_2.5.0            
#> [10] plyr_1.8.6           sys_3.4              stats4_4.1.0        
#> [13] evaluate_0.14        ggplot2_3.3.3        highr_0.9           
#> [16] pillar_1.6.1         rlang_0.4.11         caret_6.0-88        
#> [19] data.table_1.14.0    rpart_4.1-15         Matrix_1.3-3        
#> [22] rmarkdown_2.8        splines_4.1.0        gower_0.2.2         
#> [25] stringr_1.4.0        munsell_0.5.0        diagram_1.6.5       
#> [28] compiler_4.1.0       xfun_0.23            pkgconfig_2.0.3     
#> [31] askpass_1.1          shape_1.4.6          htmltools_0.5.1.1   
#> [34] nnet_7.3-16          openssl_1.4.4        tidyselect_1.1.1    
#> [37] tibble_3.1.2         prodlim_2019.11.13   codetools_0.2-18    
#> [40] fansi_0.5.0          crayon_1.4.1         dplyr_1.0.6         
#> [43] withr_2.4.2          ModelMetrics_1.2.2.2 MASS_7.3-54         
#> [46] recipes_0.1.16       grid_4.1.0           nlme_3.1-152        
#> [49] gtable_0.3.0         lifecycle_1.0.0      DBI_1.1.1           
#> [52] magrittr_2.0.1       pROC_1.17.0.1        credentials_1.3.0   
#> [55] scales_1.1.1         cli_2.5.0.9000       stringi_1.6.2       
#> [58] reshape2_1.4.4       doParallel_1.0.16    timeDate_3043.102   
#> [61] ellipsis_0.3.2       generics_0.1.0       vctrs_0.3.8         
#> [64] lava_1.6.9           tools_4.1.0          glue_1.4.2          
#> [67] purrr_0.3.4          parallel_4.1.0       survival_3.2-11     
#> [70] yaml_2.2.1           colorspace_2.0-1     knitr_1.33
```
