---
title: "STATS 531 Final Project"
date: "April 28, 2016"
output: html_document
---







\newcommand\prob{\mathbb{P}}
\newcommand\E{\mathbb{E}}
\newcommand\var{\mathrm{Var}}
\newcommand\cov{\mathrm{Cov}}
\newcommand\loglik{\ell}
\newcommand\R{\mathbb{R}}
\newcommand\data[1]{#1^*}
\newcommand\params{\, ; \,}
\newcommand\transpose{\scriptsize{T}}
\newcommand\eqspace{\quad\quad}
\newcommand\myeq[1]{\eqspace \displaystyle #1}
\newcommand\lik{\mathscr{L}}
\newcommand\loglik{\ell}
\newcommand\profileloglik[1]{\ell^\mathrm{profile}_#1}
\newcommand\ar{\phi}
\newcommand\ma{\psi}
\newcommand\AR{\Phi}
\newcommand\MA{\Psi}
\newcommand\ev{u}
\newcommand\given{{\, | \,}}
\newcommand\equals{{=\,}}
\newcommand\matA{\mathbb{A}}
\newcommand\matB{\mathbb{B}}
\newcommand\matH{\mathbb{H}}
\newcommand\covmatX{\mathbb{U}}
\newcommand\covmatY{\mathbb{V}}

#### 1) Goal: Produce brain network using raw fMRI dataset

#### 2) Background: 

Brain networks have been used as a tool for modelling the brain's structural or functional connectome [1,2]. To estimate a brain network, we first gather raw functional magnetic resonance imaging (fMRI) time series data. After fMRI pre-processing steps, voxel-level time series data is averaged within each brain region of interest, yielding a region-level time series data. 

The most common way to produce a brain network from this fMRI data is to use Pearson Correlation coefficient. Pearson Correlation coefficients between pairs of time series of a single brain region are calculated and used as weights for a weighted adjacency matrix. 

My goal for the project is to explore another method (other than calculating correlation coefficients) to estimate a brain network from fMRI data which can effectively incorporate spatial and temporal dependence between the time series data. 

#### 3) Data:

I will use a region-level fMRI time series data downloaded from http://biovis.net/year/2014/info/contest_data. Under the section 'Contest subject networks', there are time series data derived from 18 subjects. Each time series is constructed from a 15-minute scan. 

For the project, I will use a time series data of a single subject chosen at random. I will denote this region-level time series data as $Y^*_{1:T}$. Here, $Y^*_n\in \mathbb{R}^r$ where $r=167$ is the number of brain regions of interest and $T$ is the number of time points.

#### 4) Data Analysis

##### 1. LG-POMP

Following the midterm project, I first use LG-POMP model to fit the dataset. 

[Process] $\eqspace X_{n} = \matA X_{n-1} + \epsilon_n$, $\eqspace \epsilon_n\sim N[0,\covmatX]$,

[Measurement] $\eqspace Y_{n} = \matB X_n + \eta_n$, $\eqspace \eta_n\sim N[0,\covmatY]$,

where $\matA, \matB, \covmatX$, and $\covmatY$ are $r\times r$ matrices.


Model is fitted using Kalman filter and EM algorithm. The code is written in matlab (EM_Kalman_filter.m). Result can be found in LGPOMP.mat.

Below, I have done simulation based on fitted $\matA, \matB, \covmatX$, and $\covmatY$.

<img src="figure/Final-LGPOMP-1.png" title="plot of chunk LGPOMP" alt="plot of chunk LGPOMP" style="display: block; margin: auto;" />


##### 2. Dynamic Causal Modeling

However, in order to make inference on underlying neuronal states, above LG-POMP model is inappropriate. It is because the observed fMRI response to change in neuronal states can be delayed by few seconds [5].

Dynamic causal modelling was first introduced by [6] and it describes the change in neuronal states (x) in response of external stimulation (u). Together with haemodynamic model [3,7] which describes relationship between neuronal state (x) and observed fMRI (y), we can settle the following model [4]:


[neuronal state] $\dot{x} = Ax + \sum^{m}_{j=1}u_jB^{(j)}x+Cu$,

[signal] $\dot{s} = x - \kappa_s s - \kappa_f (e^f - 1)$,

[blood flow] $\dot{f} = s e^{-f}$,

[volume] $\dot{v} = \frac{1}{\tau_0}(e^f - e^{v/\alpha})e^{-v}$,

[deoxyhemoglobin] $\dot{q} = \frac{1}{\tau_0}(\frac{e^{f-q}(1-(1-E_0)^{e^{-f}})}{E_0}-e^{(1-\alpha)v/\alpha})$,

[observed BOLD] $y^* = V_0(4.3\nu_0E_0TE(1-e^q) + \epsilon_0r_0E_0TE(1-e^{q-v}) + (1-\epsilon_0)(1-e^v))$. 

Details on parameters and thier prior values can be found in [4,7]. 

Most importantly, matrix $A$ represents the baseline connectivity without external stimulus among brain regions. It shows causal relationship thus used to infer 'effective connectivity' network of the brain. 


Below, I fit this model using time series of a single brain region, region 1, i.e., $y^*_n=Y^*_n[1]$. Since dataset contains resting state fMRI, there is no external stimulus , i.e., $u=0$. Moreover, I add stochasticity to the model. 


```r
fMRI<-data.frame(t(as.matrix(read.table(file="1X4I9DF0_time_series.txt",header=FALSE))))
time=seq(0,15,length.out=1200)*60  

brain_data <- data.frame(cbind(region1=fMRI[801:950,1],time=time[1:150]))
```



```r
brain_statenames <- c("x","s","f","v","q")
brain_paramnames <- c("a","k_s","k_f","tao0","alpha","E0","V0","nu0","TE","r0","eps0","sigma","rho")
(brain_obsnames <- colnames(brain_data)[1])
```

```
## [1] "region1"
```



```r
brain_dmeasure <- "
  lik = dnorm(region1,V0*(4.3*nu0*E0*TE*(1-exp(q))+eps0*r0*E0*TE*(1-exp(q-v))+(1-eps0)*(1-exp(v))),sigma,give_log);
"

brain_rmeasure <- "
  region1 = rnorm(V0*(4.3*nu0*E0*TE*(1-exp(q))+eps0*r0*E0*TE*(1-exp(q-v))+(1-eps0)*(1-exp(v))),sigma);
"


brain_rprocess <- "
  double dW = rnorm(0,sqrt(dt));
  x += a*x*dt+rho*dW;
  s += (x - k_s*s - k_f*(exp(f)-1))*dt;
  f += (s*exp(-f))*dt;
  v += ((exp(f)-exp(v/alpha))*exp(-v)/tao0)*dt;
  q += (((exp(f-q)*(1-pow(1-E0,exp(-f)))/E0) - exp((1-alpha)*v/alpha))/tao0)*dt;
"

brain_fromEstimationScale <- "
 Tk_s = exp(k_s);
 Tk_f = exp(k_f);
 Ttao0 = exp(tao0);
 TV0 = exp(V0);
 Tnu0 = exp(nu0);
 TTE = exp(TE);
 Tr0 = exp(r0);
 Teps0 = exp(eps0);
 Tsigma = exp(sigma);
 Trho = exp(rho);
"

brain_toEstimationScale <- "
 Tk_s = log(k_s);
 Tk_f = log(k_f);
 Ttao0 = log(tao0);
 TV0 = log(V0);
 Tnu0 = log(nu0);
 TTE = log(TE);
 Tr0 = log(r0);
 Teps0 = log(eps0);
 Tsigma = log(sigma);
 Trho = log(rho);
"


brain_initializer <- "
 x=-0.01;
 s=-0.01;
 f=-0.02;
 v=-0.01;
 q=0.01;
"
```

Below plot shows the BOLD signal of region 1, which is my dataset. Here, unit of time is a second.


```r
brain2 <- pomp(
  data=brain_data,
  times="time",
  t0=0,
  rprocess=euler.sim(
    step.fun=Csnippet(brain_rprocess),
    delta.t=0.01
  ),
  rmeasure=Csnippet(brain_rmeasure),
  dmeasure=Csnippet(brain_dmeasure),
  fromEstimationScale=Csnippet(brain_fromEstimationScale),
  toEstimationScale=Csnippet(brain_toEstimationScale),
  obsnames = brain_obsnames,
  statenames=brain_statenames,
  paramnames=brain_paramnames,
  initializer=Csnippet(brain_initializer)
)
plot(brain2)
```

<img src="figure/Final-pomp_brain-1.png" title="plot of chunk pomp_brain" alt="plot of chunk pomp_brain" style="display: block; margin: auto;" />

For the local search, I set run level to 1. 


```r
run_level <- 1
switch(run_level,
       {brain_Np=10000; brain_Nmif=10; brain_Neval=10; brain_Nglobal=10; brain_Nlocal=10}, 
       {brain_Np=20000; brain_Nmif=50; brain_Neval=10; brain_Nglobal=10; brain_Nlocal=10}, 
       {brain_Np=60000; brain_Nmif=300; brain_Neval=10; brain_Nglobal=100; brain_Nlocal=20}
)
```

Equations describing the model can become unstable depending on the values of parameters. Thus, I fix three parameters' values here at their prior means [4,7]. 


```r
brain_mle <- c(a=-1,k_s=1.1635457,k_f=0.3076257,tao0=4.081991,alpha=0.32,E0=0.34,V0=123.94151,nu0=2.4213849,TE=0.03129477,r0=3.4150168,eps0=9.992907,sigma=1.627199,rho=0.04998143)
bsflu_fixed_params <- c(a=-1,alpha=0.32,E0=0.34)
```

Here is an example of simulation from a set of parameters. Again, unit of time is a second. 


```r
simulation <- simulate(brain2,params=brain_mle)
plot(simulation)
```

<img src="figure/Final-brain_sim-1.png" title="plot of chunk brain_sim" alt="plot of chunk brain_sim" style="display: block; margin: auto;" />

I have done basic particle filtering first. 


```r
pf <- pfilter(brain2,params=brain_mle,Np=10000)
plot(pf)
```

<img src="figure/Final-brain_pfilter-1.png" title="plot of chunk brain_pfilter" alt="plot of chunk brain_pfilter" style="display: block; margin: auto;" />



```r
require(doParallel)
cores <- 20   
registerDoParallel(cores)
mcopts <- list(set.seed=TRUE)

set.seed(396658101,kind="L'Ecuyer")
```

Here, I used smaller rw.sd for the parameters $\kappa_f$, sigma (measurement error), and rho (process error), since I found that their values can also make the equation unstable. 


```r
brain_cooling.fraction.50 <- 0.5

stew(file=sprintf("R1local_search-%d.rda",run_level),{
  
  t_local <- system.time({
    mifs_local <- foreach(i=1:brain_Nlocal,.packages='pomp', .combine=c, .options.multicore=mcopts) %dopar%  {
      mif2(
        brain2,
        start=brain_mle,
        Np=brain_Np,
        Nmif=brain_Nmif,
        cooling.type="geometric",
        cooling.fraction.50=brain_cooling.fraction.50,
        transform=TRUE,
        rw.sd=rw.sd(
          k_s=0.02,
          k_f=0.002,
          tao0=0.02,
          V0=0.02,
          nu0=0.02,
          TE=0.02,
          r0=0.02,
          eps0=0.02,
          sigma=0.002,
          rho=0.0002
        )
      )
      
    }
  })
  
},seed=900242057,kind="L'Ecuyer")
```



```r
stew(file=sprintf("R1lik_local-%d.rda",run_level),{
    t_local_eval <- system.time({
    liks_local <- foreach(i=1:brain_Nlocal,.packages='pomp',.combine=rbind) %dopar% {
      evals <- replicate(brain_Neval, logLik(pfilter(brain2,params=coef(mifs_local[[i]]),Np=brain_Np)))
      logmeanexp(evals, se=TRUE)
    }
  })
},seed=900242057,kind="L'Ecuyer")

results_local <- data.frame(logLik=liks_local[,1],logLik_se=liks_local[,2],t(sapply(mifs_local,coef)))
summary(results_local$logLik,digits=5)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
## -471.97 -422.59 -410.36 -418.41 -407.24 -400.41
```


```r
pairs(~logLik+k_s+k_f+tao0,data=subset(results_local,logLik>max(logLik)-50))
```

<img src="figure/Final-pairs_local-1.png" title="plot of chunk pairs_local" alt="plot of chunk pairs_local" style="display: block; margin: auto;" />

```r
pairs(~logLik+V0+nu0+TE,data=subset(results_local,logLik>max(logLik)-50))
```

<img src="figure/Final-pairs_local-2.png" title="plot of chunk pairs_local" alt="plot of chunk pairs_local" style="display: block; margin: auto;" />

```r
pairs(~logLik+r0+eps0+sigma+rho,data=subset(results_local,logLik>max(logLik)-50))
```

<img src="figure/Final-pairs_local-3.png" title="plot of chunk pairs_local" alt="plot of chunk pairs_local" style="display: block; margin: auto;" />

Now, let's try several different values of starting point to see the result. 


```r
brain_box <- rbind(
   k_s=c(0.2,1.3),
   k_f=c(0.23,0.41),
   tao0=c(2.5,10.3),
   V0=c(20,400),
   nu0=c(0.1,30),
   TE=c(0.01,0.45),
   r0=c(0.4,19),
   eps0=c(3,40),
   sigma=c(1.5,1.8),
   rho=c(0.0485,0.0506)
)
```

I have set run level to 2 for the global search.


```r
run_level <- 2
switch(run_level,
       {brain_Np=10000; brain_Nmif=10; brain_Neval=10; brain_Nglobal=10; brain_Nlocal=10}, 
       {brain_Np=20000; brain_Nmif=50; brain_Neval=10; brain_Nglobal=10; brain_Nlocal=10}, 
       {brain_Np=60000; brain_Nmif=300; brain_Neval=10; brain_Nglobal=100; brain_Nlocal=20}
)
```


```r
stew(file=sprintf("R1box_eval-%d.rda",run_level),{
  
  t_global <- system.time({
    mifs_global <- foreach(i=1:brain_Nglobal,.packages='pomp', .combine=c, .options.multicore=mcopts) %dopar%  mif2(
      mifs_local[[1]],
      start=c(apply(brain_box,1,function(x)runif(1,x[1],x[2])),bsflu_fixed_params)
    )
  })
},seed=1270401374,kind="L'Ecuyer")
```


```r
stew(file=sprintf("R1lik_global_eval-%d.rda",run_level),{
  t_global_eval <- system.time({
    liks_global <- foreach(i=1:brain_Nglobal,.packages='pomp',.combine=rbind, .options.multicore=mcopts) %dopar% {
      evals <- replicate(brain_Neval, logLik(pfilter(brain2,params=coef(mifs_global[[i]]),Np=brain_Np)))
      logmeanexp(evals, se=TRUE)
    }
  })
},seed=442141592,kind="L'Ecuyer")

results_global <- data.frame(logLik=liks_global[,1],logLik_se=liks_global[,2],t(sapply(mifs_global,coef)))
summary(results_global$logLik,digits=5)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
## -518.34 -468.02 -447.78 -448.92 -422.24 -387.26
```


```r
pairs(~logLik+k_s+k_f+tao0,data=results_global)
```

<img src="figure/Final-pairs_global-1.png" title="plot of chunk pairs_global" alt="plot of chunk pairs_global" style="display: block; margin: auto;" />

```r
pairs(~logLik+V0+nu0+TE,data=results_global)
```

<img src="figure/Final-pairs_global-2.png" title="plot of chunk pairs_global" alt="plot of chunk pairs_global" style="display: block; margin: auto;" />

```r
pairs(~logLik+r0+eps0+sigma+rho,data=results_global)
```

<img src="figure/Final-pairs_global-3.png" title="plot of chunk pairs_global" alt="plot of chunk pairs_global" style="display: block; margin: auto;" />


```r
plot(mifs_global)
```

<img src="figure/Final-mifs_global_plot-1.png" title="plot of chunk mifs_global_plot" alt="plot of chunk mifs_global_plot" style="display: block; margin: auto;" /><img src="figure/Final-mifs_global_plot-2.png" title="plot of chunk mifs_global_plot" alt="plot of chunk mifs_global_plot" style="display: block; margin: auto;" /><img src="figure/Final-mifs_global_plot-3.png" title="plot of chunk mifs_global_plot" alt="plot of chunk mifs_global_plot" style="display: block; margin: auto;" />

Diagnistic plot implies the possibility of problem in initial values of state variables. Except for the initial part (0~10 second region), the model seems to be well fitted to the dataset. 


#### 5) Conclusion: Limitation and Future Work

There are two main limitations of this final project. First, I have only used the dataset of a single brain region. Thus, I couldn't actually see the relationship between intrinsic neuronal states of different brain regions. Second, since equations of the model is unstable, I could not perform actual global search to find MLE. (I had to make some parameters fixed.) Thus, for future work, I am planning to extend and apply this model to fit the dataset of more than one brain regions with more care of handling parameter space.

In addition, I would also like to work more on the initial values of state variables, and try to run the code under higher run level. 

Note: Here, I used DCM to model the observed fMRI signal following [5]. However, there are also papers that describes the model in terms of 'observed local BOLD changes' (denoted as $\frac{S-S_0}{S_0}$) [4]. Using the observed BOLD signal was more sensible for my dataset (gave better fit), however, I should study deeper to get this straight. 

#### 6) References

[1] E. T. Bullmore and D. S. Bassett. Brain Graphs: Graphical Models of the Human Brain Connectome. *Annual Review of Clinical Psychology*, 7:113-40, 2011.

[2] E. T. Bullmore and O. Sporns. Complex brain networks: graph theoretical analysis of structural and functional systems. *Nature Reviews Neuroscience*, 10:186-198, 2009.

[3] R.B. Buxton, E.C. Wong, and LR. Frank. Dynamics of blood flow and oxygenation changes during brain activation: the Balloon model. *Magn. Reson. Med.* 39:855-864, 1998.

[4] J. Daunizeau, L. Lemieux, A. E. Vaudano, K. E. Stephan. An electrophysiological validation of stochastic DCM for fMRI. *Front Comput Neurosci.*, 6:103. doi: 10.3389/fncom.2012.00103. eCollection 2012.

[5] K. Friston. Causal modelling and brain connectivity in functional magnetic resonance imaging. *PLoS Biol*, 7(2):e1000033. doi:10.1371/journal.pbio.1000033, 2009.

[6] K. J. Friston, L. Harrison, W. Penny. Dynamic causal modelling. *Neuroimage*, 19:1273-1302, 2003.

[7] D. E. Glaser, K. J. Friston, A. Mechelli, R. Turner, and C. J. Price. Haemodynamic modelling. *Human Brain Function*, Edited by: Frackowiak RSJ, San Diego:Elsevier, 823-842, 2003. 

[8] A. A. King, D. Nguyen, and E. L. Ionides. Statistical Inference for Partially Observed Markov Processes via the R Package pomp. *Journal of Statistical Software*, 69(12) doi:10.18637/jss.v069.i12, 2016.

[9] A. C. Marreiros, K E. Stephan, and K. J. Friston. Dynamic causal modeling. *Scholarpedia*, 5(7):9568, 2010.

[10] K. E. Stephan, N. Weiskopf, P. M. Drysdale, P. A. Robinson, and K. J. Friston. Comparing hemodynamic models with DCM. *NeuroImage*, 38:387-401, 2007.

[11] K. J. Worsley, C. Liao, J. Aston, V. Petre, G. Duncan, F. Morales, and A. Evans. A general statistical analysis for fMRI data. *Neuroimage*, 15(1):1-15, 2002.


For the code, I refer to http://ionides.github.io/531w16/#class-notes.
