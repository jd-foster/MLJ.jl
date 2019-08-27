# Getting Started

#### [Installation instructions](https://github.com/alan-turing-institute/MLJ.jl/blob/master/README.md)

#### [Cheatsheet](mlj_cheatsheet.md)

```@repl doda
import Random.seed! # hide
seed!(1234) # hide
```

### Plug-and-play model evaluation

To load some data install the
[RDatasets](https://github.com/JuliaStats/RDatasets.jl) in your load
path and enter

```@repl doda
using RDatasets
iris = dataset("datasets", "iris"); # a DataFrame
```
and then split the data into input and target parts:

```@repl doda
X = iris[:, 1:4];
y = iris[:, 5];
```

In MLJ a *model* is a struct storing the hyperparameters of the learning
algorithm indicated by the struct name.  Assuming the DecisionTree
package is in your load path, we can instantiate a
DecisionTreeClassifier model like this:

```@repl doda
using MLJ
@load DecisionTreeClassifier verbosity=1
tree_model = DecisionTreeClassifier(max_depth=2)
```

*Important:* DecisionTree and most other packages implementing machine
learning algorithms for use in MLJ are not MLJ dependencies. If such a
package is not in your load path you will receive an error explaining
how to add the package to your current environment.

Once loaded, a model is evaluated with the `evaluate` method:

```@repl doda
evaluate(tree_model, X, y, 
         resampling=CV(shuffle=true), measure=cross_entropy, verbosity=0)
```

Evaluating against multiple performance measures is also possible. See
[Evaluating model performance](evaluating_model_performance.md) for details.


### Training and testing by hand

Wrapping the model in data creates a *machine* which will store
training outcomes:

```@repl doda
tree = machine(tree_model, X, y)
```

Training and testing on a hold-out set:

```@repl doda
train, test = partition(eachindex(y), 0.7, shuffle=true); # 70:30 split
fit!(tree, rows=train);
yhat = predict(tree, X[test,:]);
yhat[3:5]
cross_entropy(yhat, y[test]) |> mean
```

Notice that `yhat` is a vector of `Distribution` objects (because
DecisionTreeClassifier makes probabilistic predictions). The methods
of the [Distributions](https://github.com/JuliaStats/Distributions.jl)
package can be applied to such distributions:

```@repl doda
broadcast(pdf, yhat[3:5], "virginica") # predicted probabilities of virginica
mode.(yhat[3:5])
```

One can explicitly get modes by using `predict_mode` instead of `predict`:

```@repl doda
predict_mode(tree, rows=test[3:5])
```

Machines have an internal state which allows them to avoid redundant
calculations when retrained, in certain conditions - for example when
increasing the number of trees in a random forest, or the number of
epochs in a neural network. The machine building syntax also
anticipates a more general syntax for composing multiple models, as
explained in [Learning Networks](learning_networks.md).

There is a version of `evaluate` for machines as well as models:

```@repl doda
evaluate!(tree, resampling=Holdout(fraction_train=0.5, shuffle=true),
                measure=cross_entropy,
                verbosity=0)
```
Changing a hyperparameter and re-evaluating:

```@repl doda
tree_model.max_depth = 3
evaluate!(tree, resampling=Holdout(fraction_train=0.5, shuffle=true),
          measure=cross_entropy,
          verbosity=0)
```

### Next steps

To learn a little more about what MLJ can do, take the MLJ
[tour](https://github.com/alan-turing-institute/MLJ.jl/blob/master/examples/tour/tour.ipynb),
and then return to the manual as needed. Read at least the remainder
of this page before considering serious use of MLJ.


### Prerequisites

MLJ assumes some familiarity with the `CategoricalValue` and
`CategoricalString` types from
[CategoricalArrays.jl](https://github.com/JuliaData/CategoricalArrays.jl),
used here for representing categorical data. For probabilistic
predictors, a basic acquaintance with
[Distributions.jl](https://github.com/JuliaStats/Distributions.jl) is
also assumed.


### Data containers and scientific types

The MLJ user should acquaint themselves with some
basic assumptions about the form of data expected by MLJ, as outlined
below. 

```
machine(model::Supervised, X, y) 
machine(model::Unsupervised, X)
```

Each supervised model in MLJ declares the permitted *scientific type*
of the inputs `X` and targets `y` that can be bound to it in the first
constructor above, rather than specifying specific machine types (such
as `Array{Float32, 2}`). Similar remarks apply to the input `X` of an
unsupervised model. Scientific types are julia types defined in the
package
[ScientificTypes.jl](https://github.com/alan-turing-institute/ScientificTypes.jl),
which also defines the convention used here (and there called *mlj*)
for assigning a specific scientific type (interpretation) to each
julia object (see the `scitype` examples below).

The basic "scalar" scientific types are `Continuous`, `Multiclass{N}`,
`OrderedFactor{N}` and `Count`. Be sure you read [Container element
types](@ref) below to be guarantee your scalar data is interpreted
correctly. Most containers also have a scientific
type.


![](scitypes.png)

```@repl doda
scitype(4.6)
scitype(42)
x1 = categorical(["yes", "no", "yes", "maybe"]);
scitype(x1)
X = (x1=x1, x2=rand(4), x3=rand(4))  # a "column table"
scitype(X)
```

#### Tabular data

All data containers compatible with the
[Tables.jl](https://github.com/JuliaData/Tables.jl) interface (which
includes all source formats listed
[here](https://github.com/queryverse/IterableTables.jl)) have the
scientific type `Table{K}`, where `K` depends on the scientific types
of the columns, which can be individually inspected using `schema`:

```@repl doda
schema(X)
```

*Figure 1. Part of the scientific type heirarchy in* ScientificTypes.jl.


#### Inputs

Since an MLJ model only specifies the scientific type of data, if that
type is `Table` - which is the case for the majority of MLJ models -
then any Tables.jl format is permitted. However, the Tables.jl API
excludes matrices. If `Xmatrix` is a matrix, convert it to a column
table using `X = MLJ.table(Xmatrix)`.

Specifically, the requirement for an arbitrary model's input is `scitype(X)
<: input_scitype(model)`.


#### Targets

The target `y` expected by MLJ models is generally an
`AbstractVector`. A multivariate target `y` will generally be a vector of
*tuples*. The tuples need not have uniform length, so some forms of
sequence prediction are supported. Only the element types of `y`
matter (the types of `y[j]` for each `j`). 

Specifically, the type requirement for a model target is `scitype(y) <:
target_scitype(model)`.

#### Querying a model for data types 

One can inspect the admissible scientific types of a model's input
and target. If one has a `Model` instance `model`, one can use
`scitype(model)`:

```@repl doda
tree = DecisionTreeClassifier();
scitype(tree)
```

If, however, the relevant model code has not been loaded, one can
nevertheless extract the scitypes from the model type's MLJ registry
entry:

```@repl doda
traits("DecisionTreeClassifier")
```

See also [Working with tasks](working_with_tasks.md) on searching for
models solving a specified task.


#### Container element types

Models in MLJ will always apply the *mlj* convention described
[ScientificTypes.jl](https://github.com/alan-turing-institute/ScientificTypes.jl)
to decide how to interpret the elements of your container types. Here
are the key aspects of that convention:

- Any `AbstractFloat` is interpreted as `Continuous`.

- Any `Integer` is interpreted as `Count`. 

- Any `CategoricalValue` or `CategoricalString`, `x`, is interpreted
  as `Multiclass` or `OrderedFactor`, depending on the value of
  `x.pool.ordered`.
  
- `String`s and `Char`s are *not* interpreted as `Finite`; they have
  `Unknown` scitype. Coerce vectors of strings or characters to
  `CategoricalVector`s if they represent `Multiclass` or
  `OrderedFactor` data.
  
  
- In particular, *integers* (including `Bool`s) *cannot be used to
  represent categorical data.*
  

To coerce the scientific type of a vector or table, use the `coerce`
method (re-exported from [ScientificTypes.jl](https://github.com/alan-turing-institute/ScientificTypes.jl)).







