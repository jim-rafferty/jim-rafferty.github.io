## Running Parallel Simulation Studies in Python

In my [last blog](https://jim-rafferty.github.io/2025/02/28/Simulation-Studies-in-Parallel.html) I talked about how to parallelise a for loop in `R`. This is just a quick update to show how to do the same thing in python using `joblib`. `Python` is a similar language to `R` in that it is interpreted and multithreading must be done by spawning new processes. Thankfully, `joblib` removes most of the pain to doing this that used to exist, but there is a little bit of syntax that is not standard and easy to forget. 

First, library imports and data generating function

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

import statsmodels.formula.api as smf

def gen_data(N, params):

    rng = np.random.default_rng(params["seed"])
    
    X = rng.normal(loc=0.0, scale=1.0, size=N)
    rand_group = rng.integers(
	    low=0, 
		high=params["n_groups"], 
		size=N
	)

    random_intercept = rng.normal(
	    loc=0.0, 
	    scale=params["intercept_sd"],
	    size=params["n_groups"]
	)
    random_slope = rng.normal(
	    loc=0.0, 
	    scale=params["slope_sd"], 
	    size=params["n_groups"]
	)
    noise = rng.normal(loc=0.0, scale=params["noise_sd"], size=N)

    response = (params["beta0"] + random_intercept[rand_group] +
               (params["beta1"] + random_slope[rand_group]) * X + 
                noise)
   
    return pd.DataFrame({
      "response":response,
      "X":X,
      "group":rand_group,
      "random_intercept":random_intercept[rand_group],
      "random_slope":random_slope[rand_group]
    })
```


Running the study in series (takes 14.2 seconds).


```python
N = 5000
params = {
    "seed":-1,
    "n_groups": 5,
    "beta0": 10,
    "beta1": -0.5,
    "intercept_sd": 1,
    "slope_sd": 0.25,
    "noise_sd": 1
}

n_iter = 500
bias = np.zeros(n_iter)

for idx in range(n_iter):

    params["seed"] = idx
    df = gen_data(N, params)
    model = smf.mixedlm("response ~ X", df, groups=df["group"])
    model_fit = model.fit()

    bias[idx] = params["beta1"] - model_fit.params["X"]
```


Running the study in parallel (takes 2.95 seconds):


```python
import joblib

def run_simulation_thread(N, params, idx):

    params["seed"] = idx
    df = gen_data(N, params)
    model = smf.mixedlm("response ~ X", df, groups=df["group"])
    model_fit = model.fit()
    return(params["beta1"] - model_fit.params["X"])

N = 5000
params = {
    "seed":-1,
    "n_groups": 5,
    "beta0": 10,
    "beta1": -0.5,
    "intercept_sd": 1,
    "slope_sd": 0.25,
    "noise_sd": 1
}

n_iter = 500

bias = joblib.Parallel(n_jobs=-1)( # This means use all processors
    joblib.delayed(
        run_simulation_thread
    )(
        N,
        params,
        idx
    ) for idx in range(n_iter)
)
```
