---
title : "Code examples for methodologies presented"
author: "Björn Bornkamp and Kaspar Rufibach"
date: "2020-11-02"
output: 
  html_document:
    keep_md: true
    toc: yes
    toc_depth: 2
    number_sections: true
    toc_float:
      collapsed: yes
      smooth_scroll: yes
  github_document: 
    toc: true
    toc_depth: 2
bibliography: biblio.bib
---

# Purpose

This document accompanies @bornkamp_20. The preprint is available [here](https://arxiv.org/abs/2008.05406) and the accompanying github repository is available [here](https://github.com/oncoestimand/princ_strat_drug_dev). It provides an implementation of different analysis methods for principal stratum estimands
motivated by the principal ignorability assumption. In addition an
idea for a sensitivity analysis is proposed. In this document we focus
on a time-to-event endpoint as they are currently mostly performed
(i.e., based on the Cox model). Some of the analyses presented here
would be easier to perform and/or interpret when using linear and
collapsible summary measures like the difference in restricted mean
survival time.

# Setup


```r
knitr::opts_chunk$set(echo = FALSE, cache = TRUE)
library(data.table)
library(ggplot2)
library(survival)
library(survminer)
library(mvtnorm)
library(knitr)
library(reporttools)

# function to prettify output
prettyCox <- function(mod, dig.coef = 2, dig.p = 1){
  tab1 <- data.frame(cbind(summary(mod)$conf.int), summary(mod)$coefficients)
  tab1 <- data.frame(cbind(apply(tab1[, c(5, 1)], 1:2, disp, 
        dig.coef), displayCI(as.matrix(tab1[, c(3, 4)]), digit = dig.coef), 
        formatPval(tab1[, 9], dig.p)))
  tab1 <- data.frame(cbind(rownames(summary(mod)$conf.int), tab1))
  colnames(tab1) <- c("Variable", "Coefficient", "HR = exp(coef)", "95\\% CI", "$p$-value")
  kable(tab1, row.names = FALSE)
}
```

# Generating example dataset

In this section we generate a hypothetical example dataset to
analyse. The dataset is loosely motivated by @schmid_18, which reports
results of a Phase 3 trial comparing the monoclonal antibody
Atezolizumab versus placebo (both on top of chemotherapy) in terms of
progression free survival (PFS). Survival functions of the two
treatment arms are loosely motivated by Figure 2A in that paper. We
assume that for some patients on the treatment arm
anti-drug-antibodies (ADAs) were induced by the treatment. Interest
then focuses on how presence of ADAs might impact the treatment effect
versus placebo. Note that we assume here that the ADA status is
available instantaneously after treatment start for every patient and
before any potential PFS event.  There might be events before the
intercurrent event occurs which might make observation of the
intercurrent event status impossible. Please consult the paper on how
to perform estimation in these more challenging situations.

This section can be skipped in case there is no interest in a detailed
description on how the data were generated.

We assume in this simulated example that the principal ignorability
assumption is fulfilled and that there is one main confounder, which
is a prognostic score $X$ with 4 equally likely categories ($X = 1, 2,
3, 4$). $X$ has a strong impact on the potential PFS outcomes $Y(0)$
and $Y(1)$ as well as occurence of ADA on the treatment group
$S(1)$. For all patients we simulate all potential outcomes, $Y(0)$,
$Y(1)$ and $S(1)$. Note that $S(0) \equiv 0$, i.e. ADA cannot occurr
on the control arm.


## Assess "true" effects - population dataset

In a first step and as a benchmark, we generate the "true" treatment effect for patients that are ADA+ in the treatment arm, i.e. those with $S(1) = 1$. To this end, we simulate from a very large number of patients, the "population".


```r
set.seed(123)

## define model quantities
N <- 1e6        # large sample
X_p <- 4        # number of categories of X
a <- 0.5        # governs P(S(1) = 1)

## generation of Y(1)
y1_a <- -1.2
y1_b <- 0.25
y1_c <- -0.5

## generation of Y(0)
y0_a <- -0.8
y0_b <- -0.5

# true proportion of censored observations
prob_cens <- 0.2

## "true" treatment effect for patients with S(1) = 1
## for a large dataset assuming we could measure S1 for all patients

## simulate covariate X (low X -> high disease severity)
X <- sample(1:X_p, N, prob = rep(1 / X_p, X_p), replace = TRUE)

## permute treatment assignment
Z <- sample(c(rep(0, N / 2), rep(1, N / 2)))

## generate S(1) for all patients
S1_prob <- 1 / (1 + exp(a * X))          ## with a = 0.5 --> P(S(1) = 1) ~= 0.24
S1 <- rbinom(N, 1, S1_prob)              ## S1 more likely to occur for high disease severity

## generate Y(1) for all patients, depending on Z, X and S(1)
Y1 <- rexp(N, exp(y1_a + y1_b * S1 + y1_c * X))
Y0 <- rexp(N, exp(y0_a             + y0_b * X))
Y <- Y1 * Z + Y0 * (1 - Z)

## add uniform random censoring
event <- sample(0:1, N, prob = c(prob_cens, 1- prob_cens), replace = TRUE)
Y[!as.logical(event)] <- runif(N - sum(event), 0, Y[!as.logical(event)])
adat <- data.table(Y, Z, X, S1, event)

## display model coefficients
truth_S1 <- coxph(Surv(Y, event) ~ Z + X, data = adat[S1 == 1])
truth_S0 <- coxph(Surv(Y, event) ~ Z + X, data = adat[S1 == 0])
prettyCox(truth_S1)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
Z          -0.16         0.86             [0.85, 0.86]   < 0.0001  
X          -0.52         0.59             [0.59, 0.60]   < 0.0001  

```r
prettyCox(truth_S0)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
Z          -0.41         0.66             [0.66, 0.66]   < 0.0001  
X          -0.52         0.60             [0.59, 0.60]   < 0.0001  

The results from the model coefficients are as expected: We see an overall treatment effect that is attenuated in ADA+ patients.


## Clinical trial dataset

Now we generate a clinical trial dataset from that same population, using the same approach as above.


```r
set.seed(123)

## number of patients
n <- 450
N <- 2 * n 

## simulate covariate X (low X -> high disease severity)
X <- sample(1:X_p, N, prob = rep(1 / X_p, X_p), replace = TRUE)

## permute treatment assignment
Z <- sample(c(rep(0, N / 2), rep(1, N / 2)))

## generate S(1) for all patients
S1_prob <- 1 / (1 + exp(a * X))          ## with a = 0.5 --> P(S(1) = 1) ~= 0.24
S1 <- rbinom(N, 1, S1_prob)              ## S1 more likely to occur for high disease severity

## generate Y(1) for all patients, depending on Z, X and S(1)
Y1 <- rexp(N, exp(y1_a + y1_b * S1 + y1_c * X))
Y0 <- rexp(N, exp(y0_a             + y0_b * X))
Y <- Y1 * Z + Y0 * (1 - Z)

## assume random censoring
event <- sample(0:1, N, prob = c(0.2, 0.8), replace = TRUE)
Y[!as.logical(event)] <- runif(sum(!event), 0, Y[!as.logical(event)])

S1[Z == 0] <- NA                  ## S1 not observed on control arm
dat <- data.table(Y, Z, X, S1, event)
```

# Simple analyses of clinical trial dataset

In this section we provide a quick overview of the dataset. Mimicking the "real world", the data only contains information on the time to event outcome $Y$,
whether the event occured or was censored (event), the treatment indicator $Z$ as well as whether ADAs occurred ($S$, only available on the treatment arm).

In practice more confounders $X$ would be used in the analysis. There might also be more variables (e.g. stratfication factors) that one might use for adjustment in the outcome model.


```r
## data structure (note S1 = NA on placebo)
head(dat, n = 10)
```

```
##               Y Z X S1 event
##  1:  8.96307424 1 3  0     1
##  2:  1.11133216 1 1  1     0
##  3: 27.93173650 1 3  0     1
##  4:  0.50492109 1 1  1     1
##  5:  1.30322001 1 1  0     0
##  6: 17.23821049 1 2  0     1
##  7:  2.92834128 1 4  1     0
##  8:  0.04945228 1 1  0     1
##  9:  2.24328958 1 4  0     1
## 10:  9.15981609 0 3 NA     0
```

```r
## variable S1 on treatment arm
with(dat, table(S1, X))
```

```
##    X
## S1    1   2   3   4
##   0  70  82 119  75
##   1  37  33  17  17
```

## Overall treatment effect


```r
## overall treatment effect
fit <- survfit(Surv(Y, event) ~ Z, data = dat)
ggsurvplot(fit, data = dat)
```

![](princ_strat_example_files/figure-html/plot-1.png)<!-- -->

```r
## Now assess effect of *observed* S1 on outcome on treatment arm only
fit <- survfit(Surv(Y, event) ~ S1, data = dat[Z == 1])
ggsurvplot(fit, data = dat) + 
  ggtitle("Patients on treatment arm only")
```

![](princ_strat_example_files/figure-html/plot-2.png)<!-- -->

```r
## Some simple model fits
## overall treatment effect
ov_tmt <- coxph(Surv(Y, event) ~ Z, data = dat)
prettyCox(ov_tmt)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
Z          -0.14         0.87             [0.75, 1.00]   0.05      

```r
## analysis within subgroup on treatment arm
ov_tmt_s1 <- coxph(Surv(Y, event) ~ S1, data = dat[Z == 1])
prettyCox(ov_tmt_s1)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
S1         0.12          1.13             [0.88, 1.45]   0.34      

As expected, patients with $S(1) = 1$ have a shorter PFS on the treatment arm. 

## Naive analysis vs complete placebo group

The next question is: Does $S$ also impact the treatment effect vs placebo?


```r
## naive analyses versus complete placebo group
naive <- coxph(Surv(Y, event) ~ Z, data = dat[(Z == 1 & S1 == 1) | Z == 0])
compl_placebo <- coxph(Surv(Y, event) ~ Z, data = dat[(Z == 1 & S1 == 0) | Z == 0])
prettyCox(naive)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
Z          -0.05         0.95             [0.74, 1.22]   0.69      

```r
prettyCox(compl_placebo)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
Z          -0.17         0.85             [0.72, 0.99]   0.03      

# Estimation approaches under the principal ignorability assumption

## Regression adjustment

The first and simplest approach for analysing the data motivated by
the principal ignorability assumption is to adjust for the confounders
in a regression model. Validity of this approach relies on correctly
specifying the regression model.


```r
reg_adj <- coxph(Surv(Y, event) ~ Z + X, data = dat[(Z == 1 & S1 == 1) | Z == 0])
reg_adj2 <- coxph(Surv(Y, event) ~ Z + X, data = dat[(Z == 1 & S1 == 0) | Z == 0])
prettyCox(reg_adj)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
Z          -0.15         0.86             [0.67, 1.10]   0.22      
X          -0.28         0.76             [0.69, 0.82]   < 0.0001  

```r
prettyCox(reg_adj2)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
Z          -0.18         0.83             [0.71, 0.97]   0.02      
X          -0.29         0.75             [0.70, 0.81]   < 0.0001  

## Weighting

A slightly more complex estimation approach that does not make a
parametric assumption on how to adjust for the confounder X (but an
assumption on how $S(1)$ and X are related) is to
utilize a weighting or multiple imputation approach.


```r
## fit model for ADA occurence on treatment
fit <- glm(S1 ~ X, data = dat[Z == 1], family = binomial())

## predict probability of ADA on control
wgts <- predict(fit, newdata = dat[Z == 0,], type = "response")
dat2 <- rbind(dat[Z == 1 & S1 == 1], dat[Z == 0])
wgts <- c(rep(1, nrow(dat[Z == 1 & S1 == 1])), wgts)

## analysis for treatment effect in subgroup
weighting <- coxph(Surv(Y, event) ~ Z, data = dat2, weights = wgts)
prettyCox(weighting)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
Z          -0.15         0.86             [0.63, 1.17]   0.33      

```r
## can also produce a weighted Kaplan-Meier estimate for patients with S1
km <- survfit(Surv(Y, event) ~ 1, data = dat[Z == 1 & S1 == 1])
fit1 <- approxfun(c(0, km$time), c(1, km$surv), rule = 2)

## now perform weighted prediction under placebo arm
wgts <- predict(fit, newdata = dat[Z == 0], type = "response")
cfit <- coxph(Surv(Y, event) ~ 1, data = dat[Z == 0], weight = wgts)
s0 <- survfit(Surv(Y, event) ~ 1, data = dat[Z == 0], weight = wgts)
fit0 <- approxfun(c(0, s0$time), c(1, s0$surv), rule = 2)
s0_2 <- survfit(Surv(Y, event) ~ 1, data = dat[Z == 0])
fit0_2 <- approxfun(c(0, s0_2$time), c(1, s0_2$surv), rule = 2)
curve(fit1(x), 0, 30, lwd = 2, ylim = c(0, 1), xlab = "Time (Months)",
      main = "Kaplan-Meier Estimate", ylab="Survival Probability")
curve(fit0(x), add = TRUE, col = 2, lwd = 2)
curve(fit0_2(x), add = TRUE, col = 3, lwd = 2)
legend("topright", legend = c("Z = 1, S(1) = 1", "Z = 0 (weighted by P(S(1) = 1))", "Z = 0 (all)"),
       col = 1:3, lwd = 2)
```

![](princ_strat_example_files/figure-html/Weighting approaches-1.png)<!-- -->

```r
## treatment effect in complement
wgts <- 1-predict(fit, newdata = dat[Z == 0,], type = "response")
dat2 <- rbind(dat[Z == 1 & S1 == 0], dat[Z == 0])
wgts <- c(rep(1, nrow(dat[Z == 1 & S1 == 0])), wgts)
tmt_compl <- coxph(Surv(Y, event) ~ Z, data = dat2, weights = wgts)
prettyCox(tmt_compl)
```



Variable   Coefficient   HR = exp(coef)   95\% CI        $p$-value 
---------  ------------  ---------------  -------------  ----------
Z          -0.14         0.87             [0.74, 1.03]   0.10      

## Multiple imputation

The analysis using weighting does not account for uncertainty in estimated weights. An alternative is multiple imputation.



```r
## multiple imputation approach
cf <- coef(fit)
cov_mat <- vcov(fit)

nSim <- 200
sim_pars <- rmvnorm(nSim, cf, cov_mat)
impute <- function(x, par){
  linpred <- par[1] + par[2] * x
  prob <- 1 / (1 + exp(-linpred))
  rbinom(length(linpred), 1, prob)
}

dat_trt0 <- dat[Z == 0, ]
dat_trt1 <- dat[Z == 1, ]

loghr <- loghr_vr <- matrix(nrow = nSim, ncol = 2)
for(i in 1:nSim){
  ADA_imp <- impute(dat_trt0$X, sim_pars[i, ])
  dat_trt0i <- dat_trt0
  dat_trt0i$S1 <- ADA_imp
  dat_imp <- rbind(dat_trt1, dat_trt0i)
  fit1 <- coxph(Surv(Y, event) ~ Z, data = dat_imp[S1 == 1, ])
  fit0 <- coxph(Surv(Y, event) ~ Z, data = dat_imp[S1 == 0, ])
  loghr[i,] <- c(coef(fit1), coef(fit0))
  loghr_vr[i,] <- c(vcov(fit1), vcov(fit0))
}

## point estimate
colMeans(loghr)
```

```
## [1] -0.1405004 -0.1425921
```

```r
exp(colMeans(loghr))
```

```
## [1] 0.8689233 0.8671077
```

```r
vr1 <- mean(loghr_vr[, 1]) + var(loghr[, 1])
vr2 <- mean(loghr_vr[, 2]) + var(loghr[, 2])
sqrt(vr1); sqrt(vr2)
```

```
## [1] 0.193235
```

```
## [1] 0.09006709
```

```r
mi <- c(colMeans(loghr), sqrt(vr1), sqrt(vr2))
```

## Summary of various estimators

Multiple imputation point estimate is similar to weighting, but now with larger standard errors.


```r
approach <- c("Truth", "Naive", "Regression adjustment", "Weighting", "Multiple imputation")
est <- c(coef(truth_S1)[1], coef(naive)[1], coef(reg_adj)[1], coef(weighting), mi[1])
se <- c(NA, sqrt(vcov(naive)), sqrt(vcov(reg_adj)[1, 1]), sqrt(vcov(weighting)[1, 1]), mi[3])
tab <- data.table(approach = approach, estimate = round(est, 3), se = round(se, 3), "hazard ratio" = round(exp(est), 3))
kable(tab)
```



approach                 estimate      se   hazard ratio
----------------------  ---------  ------  -------------
Truth                      -0.156      NA          0.855
Naive                      -0.050   0.125          0.951
Regression adjustment      -0.154   0.126          0.858
Weighting                  -0.153   0.159          0.858
Multiple imputation        -0.141   0.193          0.869

As an overall conclusion, results using both regression adjustment as well as weighting
approaches point towards the fact that the treatment effect in the
subgroup for which $S(1)=1$ is a bit larger (smaller log-HR) and closer to
the true effect, than compared to the effect based on the naive
analysis versus the complete placebo group. This is because patients
with $S(1)=1$ generally have a worse prognosis (independent on which
treatment group they are).

# Sensitivity analysis

As discussed in @bornkamp_20, the principal ignorability assumption
is unverifiable. In this section we propose a
sensitivity analysis. The essential information missing for the
analysis of interest is how $Y(0) | S(1) = 1$ and $Y(0) | S(1) = 0$ are related to
each other, i.e. what impact does $S(1)$ have on $Y(0)$. As $S(1)$ and $Y(0)$
are not jointly observed this is unknown.

Here we propose a sensitivity analysis where we impute $S(1)$ on the
placebo arm. This will result in a specific HR quantifiying the effect that $S(1)$ might have on
$Y(0)$. Then we plot the HR of $S(1)$ on $Y(0)$ versus log(HR) in the subgroup of interest. 

The idea of this analysis is as follows:

+ Go randomly through different ways of assigning $S(1)$ to the control patients, so that the overall proportion of S(1) = 1 is similar to the treatment arm. 
+ For each allocation we can calculate the HR of $S(1) = 1$ vs $S(1) = 0$ on the control arm. 
+ Plot the log(HR) of treatment in $S(1) = 1$ and $S(1) = 0$ against the HR of $S(1) = 1$ vs $S(1) = 0$ on the control arm.

To cover the range of different HRs a biased sampling is used.


```r
dat_Z0 <- dat[Z == 0, ]
dat_Z1 <- dat[Z == 1, ]

sm <- sum(dat[Z == 1, ]$S1 == 1)
n1 <- nrow(dat[Z == 1, ])
n0 <- nrow(dat[Z == 0, ])

## normalized rank (used for biased sampling later)
norm_rank <- rank(dat_Z0$Y) / (n0 + 1) 
nSim <- 500
loghr <- matrix(nrow = nSim, ncol = 2)
hr <- numeric(nSim)
for(i in 1:nSim){
  pp <- rbeta(1, sm + 1 / 3, n1 - sm + 1 / 3) ## sample from posterior for P(S1 = 1) with uniformative prior
  
  ## now randomly pick who would have S(1) = 1 in the control group to
  ## efficiently explore also "extreme" allocations sometimes favor
  ## patients with long (or short) event (or censoring) time to have
  ## S(1) = 1.
  ind <- sample(0:1, 1)
  power <- runif(1, 0, 5)
  if(ind == 0){probs <- norm_rank ^ power}
  if(ind == 1){probs <- (1 - norm_rank) ^ power}
  sel <- sample(1:n0, pp * n0, prob = probs)
  no_sel <- setdiff(1:n0, sel)
  S1_imp <- rep(0, n0)
  S1_imp[sel] <- 1
  
  ## this calculates the impact S(1) has on Y(0) for this imputation
  ## measured in terms of the log-hazard ratio
  hr[i] <- coef(coxph(Surv(Y, event) ~ S1_imp, data = dat_Z0))
  dat_Z0i <- dat_Z0
  dat_Z0i$S1 <- S1_imp
  dat_imp <- rbind(dat_Z1, dat_Z0i)
  fit1 <- coxph(Surv(Y, event) ~ Z, data = dat_imp[S1 == 1, ])
  fit0 <- coxph(Surv(Y, event) ~ Z, data = dat_imp[S1 == 0, ])
  loghr[i,] <- c(coef(fit1), coef(fit0))
}
fitoverall <- coxph(Surv(Y, event) ~ Z, data = dat)

plot(hr, loghr[,1], ylim = range(loghr), pch = 19,
     xlab = "log(HR) of S(1) = 1 vs S(1) = 0 on placebo arm", ylab = "log(HR) of treatment")
abline(h = 0, lty = 2)
points(hr, loghr[, 1], col = "black", pch = 19)
points(hr, loghr[, 2], col = "red", pch = 19)
abline(h = coef(fitoverall), lwd = 2, col = "blue") ## overall treatment effect
legend("bottomleft", c("S(1) = 1", "S(1) = 0", "Overall"), col = c("black", "red", "blue"),
       pch = c(19, 19, -1), lwd = c(-1, -1, 2), title = "Population", bg = "white")
```

![](princ_strat_example_files/figure-html/Sensitivity analysis-1.png)<!-- -->

From this plot it can be seen that the treatment effect in the
subpopulation with $S(1) = 1$ depends quite strongly on how much
$S(1)$ impacts $Y(0)$. The observed impact of $S(1)$ on $Y(1)$
(unadjusted for $X$) lead to a log-HR of around 0.28 so that one might
speculate that the impact of $S(1)$ on $Y(0)$ would be smaller than
that. In this area from 0 to 0.28 one can still see that one would
observe a benefit for the treatment group in the subgroup with $S(1)=1$.

# References
