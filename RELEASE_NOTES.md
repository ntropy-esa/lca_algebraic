# 0.0.14

* Internal change for generated webapp 
* Add several display options to graphs()
* Fix overlapping graph bug

# 0.0.13 

* Fixed bug in param **extract_activities** of **multiLCAAlgebric**

# 0.0.12

* Added a step of simplification of models in **sobol_simplify_model** by removing minor terms of sums
* Added support for alternate names for impact methods with the function **set_custom_impact_labels**
* Added parameter **extract_activities** to **multiLCAAlgebric** in order to compute contribution of a subset of activities to the global impact 

# 0.0.11

Added **BETA** distribution type. See [online doc](https://oie-mines-paristech.github.io/lca_algebraic/doc/params.html#lca_algebraic.params.DistributionType)

# 0.0.10

Fixed [issue #4](https://github.com/oie-mines-paristech/lca_algebraic/issues/4) : Added back handling of default params

# 0.0.9 

* Removed cache for **find{Bio/Tech}Act**, in order to lower memory usage.
Instead, we now use core Brightway capabilities for searching activities.
* Added function **sobol_simplify_model** for generating simplified models, 
and **compare_simplified** for comparing their distributions with the initial model.

# 0.0.8

Cleanup dependencies

# 0.0.7

* Added parameters 'var_params' for all stats tools, for providing list of parameters to vary.
  Other parameters will be set at their default value.
* Fixed typo in the function name **incer_stochastic_dahboard** => **incer_stochastic_dashboard**
* Fixed bug of int overload : convert everything to float

# 0.0.6 

* Rename **modelToExpr()** to **simplifiedModel()**
* Improve performance of computation. Added multithread computation for LCA and Sobol.
* Added method **sobol_simplify_model** that computes list of most meaningful parameters, 
based on Sobol analysis, and then generates simplified models based on it.

# 0.0.5

* Add function `modelToExpr(model, impacts)`, returning expression with background activities replaced 
by the value of each impact.


# 0.0.4

* Added button for **exporting** data as csv on any **Dataframe** in stats tools
* **actToExpression** now replaces fixed params by their default values, providing simplified models.
* **newSwitchAct** can take custom amounts (not only '1') by providing tuple (act, amount)

# 0.0.3

First released version