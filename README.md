---
jupyter:
  jupytext:
    formats: ipynb,Rmd
    text_representation:
      extension: .Rmd
      format_name: rmarkdown
      format_version: '1.2'
      jupytext_version: 1.3.2
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

<!-- #region -->
# Introduction

This notebook presents **lca-algebraic** a small layer above **brightay2**, designed for the definition of **parametric inventories** with fast computation of LCA impacts, suitable for **monte-carlo** analyis.

**lca-algebraic** provides a set of  **helper functions** for : 
* **compact** & **human readable** definition of activites :  
    * search background (tech and biosphere) activities 
    * create new foreground activites with parametrized amounts
    * parametrize / update existing background activities (extending the class **Activity**)
* definition 

  
# Principles 

The main idea of this libray is to move from **procedural definition** of models (slow and prone to errors) to a **declarative / purely functionnal** definition of parametric models (models as **pure functions**). 

This enables **fast computation of LCA impacts**. 
We leverage the **power of symbolic calculus** provided by the great libary [SymPy](https://www.sympy.org/en/index.html).

We define our model in a **separate DB**, as a nested combination of : 
* other foreground activities
* background activities :
    * Technical, refering **ecoinvent DB**
    * Biopshere, refering **brightway2** biosphere activities
    
The **amounts** in exchanges are expressed either as **static amounts**, or **symbolic expressions** of pre-defined **parameters**.

Each activity of our **root model** is defined as a **parametrized combination** of the **foreground activities**, which can themselves be expressed by the **background activities**.

When computing LCA for foreground models, the library develops the model as a combination of **only background activities**. It computes **once for all** the impact of **background activities** and compiles a **fast numpy** (vectorial) function for each impact, replacing each background activity by the **static value of the corresponding impact**.

By providing **large vectors** of **parameter values** to those numpy functions, we can compute LCA for **thousands of values** at a time.

![](./doc/lca-algebraic.png)


# Compatiblity with brightway2 

Under the hood, the activities we define with **lca-algebraic** are standard **brightway2** activities. 
The amounts of exchanges are stored as **float values** or **serialized as string** in the property **formula**.

Parameters are also stored in the **brightay2** projets, making it fully compatible with **brightway**.

Thus, a model defined with **lca-algebraic** is stored as a regular **bw2** projet. We can use **bw2** native support for [parametrized dataset](https://2.docs.brightway.dev/intro.html#parameterized-datasets) for computing LCAs, even if much more slower than the method explain here.

# Doc

The followng notebook explores the main functions.
Full doc is [available here](./doc/index.html)
<!-- #endregion -->

# Install extra libraries

We need to install 3 extra libraries beforehand.<br/>
Type the following commands into your anaconda shell prompt.

**Sympy**
    
    conda install -c anaconda sympy
    
**SALib**

    conda install -c conda-forge salib
    
**slugify**

    conda install python-slugify
    


# Imports

Besides external libraries, we import several function from the local file : `lca-algebraic.py`

```{python}
# %load_ext autoreload
# %autoreload 2
import pandas as pd
import time
import matplotlib.pyplot as plt
import numpy as np
import brightway2 as bw

# Custom utils defined for inter-acv
from lca_algebraic import *
import lca_algebraic
```

# Init brightway2 and databases

```{python}
# Import Ecoinvent DB (if not already done)
# Update the PATH to suit your installation
# importDb(ECOINVENT_DB_NAME, './ecoinvent 3.4_cutoff_ecoSpold02/datasets')

# Setup bw2 and detect version of ecoinvent
initDb('bw2_seminar_2017')

# We use a separate DB for defining our foreground model / activities
USER_DB = 'Geothermal database'

# This is better to cleanup the whole foreground model each time, 
# instead of relying on a state or previous run of model.
# Any persistent state is prone to errors.
resetDb(USER_DB)

# Parameters are stored at project level : reset them also
resetParams(USER_DB)
```

<!-- #region -->
# Introduction to Numpy


Numpy is a python libray for symbolic calculus. 

You write Sympy expression as you write **standard python expressions**, using **sympy symbols** in them. 


The result is then a **symbolic expression that can be manipulated**, instead of a **numeric value**.
<!-- #endregion -->

```{python}
from sympy import symbols 

# create sympy symbol
a = symbols("a")

# Expression are not directly evaluated 
f = a * 2 + 4 
f 
```

```{python}
# symbols can be replaced by values afterwards 
f.subs(dict(a=3))
```

In practice, you don't need to care about Sympy. Just remember that : 
* The parameters defined below are **instances of sympy symbols**
* Any **valid python expression** containing a **sympy symbol** will create a **sympy symbolic expression**


# Define parameters

We define the parameters of the model.

The numeric parameters are **instances of sympy 'Symbol'**. 

Thus, any python arithmetic expression composed of parameters will result in a **symbolic expression** for later evaluation, instead of a numeric result.

```{python}
# Example of 'float' parameters

a = newFloatParam(
    'a', 
    default=0.5, min=0, max=2,  distrib=DistributionType.TRIANGLE, # Distribution type, linear by default
    description="hello world")

b = newFloatParam(
    'b',
    default=0.5, # Fixed if no min /max provided
    description="foo bar")

share_recycled_aluminium = newFloatParam(
    'share_recycled_aluminium',  
    default=0.6, min=0, max=1, std=0.2, distrib=DistributionType.NORMAL, # Normal distrib, with std dev
    description="Share of reycled aluminium")

# Yuo can define boolean parameters, taking only discrete values 0 or 1
bool_param = newBoolParam(
    'bool_param', 
    default=1)

# Example 'enum' parameter, acting like a switch between several possibilities
# Enum parameters are not Symbol themselves
# They are a facility to represent many boolean parameters at once '<paramName>_<enumValue>' 
# and should be used with the 'newSwitchAct' method 
elec_switch_param = newEnumParam(
    'elec_switch_param', 
    values=["us", "eu"], 
    default="us", 
    description="Switch on electricty mix")

# Another example enum param
techno_param = newEnumParam(
    'techno_param', 
    values=["technoA", "technoB", "technoC"], 
    default="technoA", 
    description="Choice of techonoly")
```

# Get references to background activities

`utils` provide two functions for easy and fast (indexed) search of activities in reference databases : 
* **findBioAct** : Search activity in **biosphere3** db
* **findTechAct** : Search activity in **ecoinvent** db

Those methods are **faster** and **safer** than using traditionnal "list-comprehension" search : 
They will **fail with an error** if **more than one activity** matches, preventing the model to be based on a random selection of one activity.


```{python}
# Biosphere activities
ground_occupuation = findBioAct('Occupation, industrial area')
heat = findBioAct('Heat, waste', categories=['air'])

# Technosphere activities

# You can add an optionnal location selector
alu = findTechAct("aluminium alloy production, AlMg3", loc="RER")
alu_scrap = findTechAct('aluminium scrap, new, Recycled Content cut-off')

# Elec 
eu_elec = findTechAct("market group for electricity, medium voltage", 'ENTSO-E')
us_elec = findTechAct("market group for electricity, medium voltage", 'US')
```

# Define the model

The model is defined as a nested combination of background activities with amounts.

Amounts are defined either as constant float values or algebric formulas implying the parameters defined above.


## Create new activities

```{python}
# Create a new activity
activity1 = newActivity(USER_DB, # We define foreground activities in our own DB
    "first foreground activity", # Name of the activity
    "kg", # Unit
    exchanges= { # We define exhanges as a dictionarry of 'activity : amount'
        ground_occupuation:3 * b, # Amount can be a fixed value 
        heat: b + 0.2  # Amount can be a Sympy expression (any arithmetic expression of Parameters)
    })

# You can create a virtual "switch" activity combining several activities with a switch parameter
elec_mix = newSwitchAct(USER_DB, 
    "elect mix", # Name
    elec_switch_param, # Sith parameter
    { # Dictionnary of enum values / activities
        "us" : us_elec,
        "eu" : eu_elec})
```

## Copy and update existing activity

You can copy and update an existing background activity.

Several new helper methods have been added to the class **Activity** for easy update of exchanges.

```{python}
alu2 = copyActivity(
    USER_DB, # The copy of a background activity is done in our own DB, so that we can safely update it                
    alu, # Initial activity : won't be altered
    "Alu2") # New name

# Update exchanges by their name 
alu2.updateExchanges({
    
    # Update amount : the special symbol *old_amount* references the previous amount of this exchange
    "aluminium, cast alloy": old_amount * (1 - share_recycled_aluminium),
    
    # Update input activity. Note also that you can use '*' wildcard in exchange name
    "electricity*": elec_mix,
    
    # Update both input activity and amount. 
    # Note that you can use '#' for specifying the location of exchange (useful for duplicate exchange names)
    "chromium#GLO" : dict(amount=4.0, input=activity1)
}) 

# Add exchanges 
alu2.addExchanges({alu_scrap :  12})
```

## Final model
The final model is just the root foreground activity referencing the others

```{python}
model = newActivity(USER_DB, "model", "kg", {
    activity1 : b * 5 + a + 1, # Reference the activity we just created
    alu2: 3 * share_recycled_aluminium, 
    alu:0.4 * a}) 
```

## Display activities

**printAct** displays the list of all exchanges of an activity.

Note that symbolic expressions have not been evaluated at this stage

```{python}
# Print_act displays activities as "pandas" tables
printAct(activity1) 
printAct(model)
```

```{python}
# You can also compute amounts by replacing parameters with a float value 
printAct(activity1, b=1.5) 
```

```{python}
# You print several activities at once to compare them
printAct(alu, alu2)
```

# Select the impacts to consider

```{python}
# List of impacts to consider
impacts = [m for m in bw.methods if 'ILCD 1.0.8 2016' in str(m) and 'no LT' in str(m)]
```

# Compute LCA

We provide two methods  for computing LCA : 
* **multiLCA** : It uses **brightway2** native parametric support. It is **much slower** and kept for **comparing results**.
* **multiLCAAlgebric** : It computes an algebric expression of the model and computes LCA once for all the background activities. Then it express each impact as a function of the parameters. This expression is then compiled into 'numpy' native code, for fast computation on vectors of samples. This version is 1 million time faster.

```{python}
# Uses brightway2 parameters
multiLCA(
    model, 
    impacts, 
                   
    # Parameters of the model
    a=1, 
    b=2, 
    elec_switch_param="us",
    share_recycled_aluminium=0.4)
```

```{python}
# Compute with algebric implementation : the values should be the same
multiLCAAlgebric(
    model, # The model 
    impacts, # Impacts
    
    # Parameters of the model
    a=1, 
    b=2,
    elec_switch_param="us",
    share_recycled_aluminium=0.4)
```

```{python}
# Here is what the symbolic model looks like :
expr, _ = actToExpression(model)
expr
```

```{python}
# You can compute several LCAs at a time :
multiLCAAlgebric(
    [alu, alu2], # The models
    
    impacts, # Impacts
    
    # Parameters of the model
    share_recycled_aluminium=0.3,
    elec_switch_param="us",
    b=4)

```

```{python}
# Fast computation for millions of separate samples
multiLCAAlgebric(
    model, # The model 
    impacts, # Impacts
    
    # Parameters of the model
    a=list(range(1, 1000000)), # All lists should have the same size
    b=list(range(1, 1000000)),
    share_recycled_aluminium=1,
    elec_switch_param="eu")
```

 # Statistic functions
 
 ## One at a time 
 
 We provide several functions for computing **statistics** for **local variations** of parameters (one at a time).
 
 ### oat_matrix(model, impacts)
 
 Shows a **matrix of impacts x parameters** colored according to the variation of the impact in the bounds of the parameter.

 


```{python}
oat_matrix(model, impacts)
```

### oat_dashboard_matrix

This functions draws a dashboard showing :
* A dropdown list, for choosing a parameter
* Several graphs of evolution of impacts for this parameter
* Full table of data
* A graph of "bars" representing the variation of each impact for this parameter (similar to the information given in oat_matrix) 

```{python}
oat_dashboard_interact(model, impacts)
```

<!-- #region -->
## Monte-carlo methods & Sobol indices

Here we leverage fast computation of monte-carlo approches. 

We compute **global sensivity analysis** (GSA).
Not only local ones.


### Sobol Matrix 

Similar to OAT matrix, we compute Sobol indices. they represent the ratio between the variance due to a given parameter and the total variance.

for easier comparison, we translate those relative sobol indices into "deviation / mean" importance :

  **relative_deviation** = **sqrt**(**sobol**(param) * **total_variance**(**impact**)) / **mean**(**impact**) 



<!-- #endregion -->

```{python}
# Show sobol indices 
incer_stochastic_matrix(model, impacts)
```

### Violin graphs

We provide a dashboard showing **violin graphs** : the exact probabilistic distribution for each impact. Together with medians of the impacts.

```{python}
incer_stochastic_violin(model, impacts)
```

### Full dashboard

A dashboard groups all this information in a single interface with tabs.

It also shows total variation of impacts. This last graph could be improved by showing stacked colored bars with the contribution of each parameter to this variation, according to Sobol indices. 

```{python}
incer_stochastic_dasboard(model, impacts)
```
