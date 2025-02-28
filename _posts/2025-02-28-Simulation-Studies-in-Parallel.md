## Running Simulation Studies in Parallel

A Simulation study is an example of a statistical task that can be very time consuming but is also a critical aspect of model development - a newly developed model must be shown to behave well with a known data generating process, and produce good estimates of parameters that would be expected in the real world so results can be trusted when the model is used on real data.

These studies typically work by fitting a model to many synthetic datasets, which programmatically can be done using a loop:
```R
for (idx in 1:n_iters) {
	data = generate_data()
	model = fit_model(data)
	estimates = extract_model_estimates(model)
}
```
This approach to running simulation studies, or any task that can be split up into many individual steps and computed in a loop, may be feasible depending on the complexity of the task or how fast the computer it is running on is, however, it is likely on modern systems to only utilise a fraction of the available performance. Modern processors generally contain many processing cores that behave like separate CPUs, and the above loop written in a language like `R` or `python` will send the usage of 1 core up to 100% while the rest sit around doing nothing!

There are lots of very sophisticated parallelisation methods that can speed up all sorts of complicated tasks but that's not typically the kind of thing that a data scientist or statistician will want to spend a lot of time implementing. We need to balance the time spent developing the analysis code against the time it takes to run, and how many times the programme needs to run (with many simulation studies only needing to be performed once!). We are looking at the low hanging fruit here - splitting up a task across multiple cores where the tasks themselves are completely independent and don't need to share any data or parameters with each other is pretty easy in most programming environments, and can lead to big improvements in performance. 

Let's look at a simple example: Suppose we have some data where there is one independent variable and one response variable. We also have a group variable and we expect there to be a linear relationship between response and independent variables. The data generating process might look like this:

$$
y_i = \beta_0 + \left(\beta_{1} + \gamma_{1j} \right) X_i + \gamma_{0j} + \epsilon_i
$$

There is a response $$y_i$$, an independent variable $$X_i$$ and a noise term $$\epsilon_i$$ for each individual $$i$$. We have fixed effects $$\beta_0$$ and $$\beta_1$$, and a random intercept $$\gamma_{0j}$$ , a random slope $$\gamma_{1j}$$ for each group $$j$$.

In code:

```R
gen_data = function(N, params)  {
  
  X = rnorm(N, mean=0, sd=1)
  rand_group = sample(1:params$n_groups, N, replace=TRUE)
  random_intercept = rnorm(params$n_groups, mean=0, sd=params$intercept_sd)
  random_slope = rnorm(params$n_groups, mean=0, sd=params$slope_sd)
  noise = rnorm(N, mean=0, sd=params$noise_sd)
  
  response = (params$beta0 + random_intercept[rand_group] +
               (params$beta1 + random_slope[rand_group]) * X + 
                noise)
  return(
    data.frame(
      response=response,
      X=X,
      group=rand_group,
      random_intercept=random_intercept[rand_group],
      random_slope=random_slope[rand_group]
    )
  )
}
```

We're going to use a linear mixed effects model to fit this data and find the bias in the estimated fixed gradient $$\beta_1$$. Firstly, a single threaded implementation of this:

```R
library(lme4) # library containing lmer
N = 5000
params = list(
  n_groups = 5,
  beta0 = 10,
  beta1 = -0.5,
  intercept_sd = 1,
  slope_sd = 0.25,
  noise_sd = 1
)

n_iter = 500
bias = numeric(n_iter)
for (idx in 1:n_iter) {
  df = gen_data(N, params)
  model = lmer(
    response ~ X + (X | group), 
    data=df
  )
  bias[idx] = params$beta1 - model@beta[2]
}
```

It's really straightforward to tell R we want to run this across multiple cores using the `foreach` and `doParallel` libraries: 
```R
library(foreach)
library(doParallel)

n_threads = detectCores() 
thread_cluster = makeCluster(n_threads)
registerDoParallel(thread_cluster, cores=n_threads)

bias = foreach(idx = 1:n_iter, .combine=cbind) %dopar% {
  library(lme4)
  df = gen_data(N, params)
  model = lmer(
    response ~ X + (X | group), 
    data=df
  )
  bias[idx] = params$beta1 - model@beta[2]
}
```

There is a bit of setup where we tell `R` we want to do a parallel computation and how many threads we want to use. You can set the number of threads to any number you like, including more threads than you have processors in your computer. This is not a good thing to do, because the thread communication carries overhead, and your computation will end up taking  more time than if you'd chosen your thread count well.

There is also another gotcha - languages like `R` are interpreted languages, and in this case the parallel implementation works by spawning multiple instances of the `R` interpreter. For some reason, in `R` the spawned instances do not remember the imported libraries from the parent process, so you need to reimport everything you will use inside the loop. 

There are some considerations to make before parallelising everything! Remember adding threads increases the memory requirements of your computation, so if you are doing something particularly memory intensive in your loop then adding threads may make your computer run out of memory. Also, some functions and models available in `R` and other environments are themselves multithreaded. If you are calling one of those functions in the loop that you're considering parallelising you might find your programme is running as efficiently as it can already.  

On my machine (M2 mac) the single threaded implementation above takes 23.3 seconds, while the parallel implementation running on all available cores takes 9.4 seconds. Not bad for a change to 5 lines. Also note that this is not a tremendously intensive example. The more time each iteration takes the better the saving will be, up to a maximum theoretical improvement of $$t_{parallel} \approx \frac{t_{serial}}{n_{threads}}$$.

Some additional resources:
- [Guidance on how to conduct simulation studies](https://pubmed.ncbi.nlm.nih.gov/16947139/)
- [`foreach` package](https://cran.r-project.org/web/packages/foreach/index.html)
- [`doParallel` package](https://cran.r-project.org/web/packages/doParallel/index.html)
