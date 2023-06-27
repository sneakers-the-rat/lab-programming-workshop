# Packaging

```{admonition} Tip
:class: tip

Packaging happens at the beginning, not the end!
```

When writing any new code, set yourself up right! 

- Make a new repository
- Initialize a package
- Sketch and create your initial structure

For now I'll assume we all know how to make a new repository and go on to making a package

## Project Structure

There is no "right way" to structure a project, but for the purpose of this workshop, let's assume it looks like this:

```
./my_project/
├── .readthedocs.yml
├── .travis.yml
├── docs
├── examples
├── my_project
│   └── __init__.py
├── pyproject.toml
├── scripts
└── tests
```

Even if your project doesn't need all of these things, we start by being explicit about where things go rather than
jumping into the code. Everything should have a place, and if it doesn't, then you should make one rather than just
making another file and putting it in the root directory. Your code isn't just code, but will have a bunch of different
components: yes you have your source directory `my_project` (or `src`, whatever you want to call it), but you also have
documentation, tests that *use* code in `my_project`, etc.

We'll return to `.readthedocs.yml` and `docs/` in [docs](docs), `.travis.yml` in [collaboration](collaboration), and
`tests/` in [tests](tests), but let's take a closer look at pyproject.toml

## `pyproject.toml`

Code is made immeasurably more useful if it can *build off other code* and *other code can build off it*.

For this to work, most contemporary programming langauges have arrived at systems for **dependency management** and 
**metadata declaration** that tells a package manager what other code is needed to run this package.

Let's take a look at the one basic example from the [prey capture repo](https://github.com/wehr-lab/Prey_Capture_Python/blob/cabd5c2aff54e4099c89802361bbe784bf5b427a/pyproject.toml)

```toml
[tool.poetry]
name = "prey_capture_python"
version = "0.1.0"
description = "python version of existing prey capture analysis pipeline"
authors = ["mshallow <c.shallow@wustl.edu>"]

[tool.poetry.dependencies]
python = ">=3.7.1,<3.10"
pandas = "^1.3.5"
matplotlib = "^3.5.1"
seaborn = "^0.11.2"
scipy = ">=1.7.0"
numpy = ">=1.21.0"
Sphinx = {version = "^4.4.0", optional = true}
furo = {version = "^2022.2.23", optional = true}
myst-parser = {version = "^0.17.0", optional = true}

[tool.poetry.dev-dependencies]

[tool.poetry.extras]
docs = ["Sphinx", "furo", "myst-parser"]

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

### Versioning

`version = "0.1.0"`

Use [semantic versioning](https://semver.org/)! There are a lot of rules, but the gist is:

* **0**.1.0 - The major version, increment when you make changes that break commonly used interfaces and use patterns
* 0.**1**.0 -  The minor version, increment when making minor (or major) changes that add functionality or are (mostly) backwards compatible
* 0.1.**0** - The patch version, increment when making bugfixes that don't change the way the code is used.

This is an indication to downstream packages and users when and what kind of changes can be made and allow them to 
pin specific versions of your code to guarantee a particular kind of functionality.

This also frees *you* up to make changes to your code, because people reliant on particular functionality *should* pin 
a particular version (or they can always roll back to one).

### Dependency Specification

There are a number of different ways to specify dependency versions that we won't cover here, but the most common ones
you'll see are

* `^0.1.0` - Caret versioning allows any version greater than that version, but less than the leftmost nonzero number. 
  So `0.1.*` are all allowed, but `0.2.0` is not.
* `>=0.1.0` - *Any* version of the code greater than that one can be used. Permissive, but has the potential to break if 
  breaking changes are introduced
* `>=0.1.0,0.3.0` - A specific *range* of versions are allowed - more specific than carat versioning.

Given these, poetry will then generate a `lockfile` that recursively computes a specific combination of packages that
satisfy the dependencies. For example, we require numpy, pandas, and scipy, but pandas and scipy both have their own requirements
for which versions of numpy they support:

`poetry show -t`

```
numpy 1.21.5 NumPy is the fundamental package for array computing with Python.
pandas 1.3.5 Powerful data structures for data analysis, time series, and statistics
├── numpy >=1.17.3
├── numpy >=1.19.2
├── numpy >=1.20.0
├── python-dateutil >=2.7.3
│   └── six >=1.5 
└── pytz >=2017.3
scipy 1.7.3 SciPy: Scientific Library for Python
└── numpy >=1.16.5,<1.23.0
```

The poetry lockfile will then compute a specific version of numpy that satisfies all those constraints.


## Metastructure

Before we put any code in our repository, we should make a diagram of the basic components that will go into it so we
have some idea of where to put it as we go! For example, for this project, we know we will have some way of loading files,
some statistical models, some geometric analysis functions, and then some functions that tie them all together.

We can use something like graphviz to keep track of that!

```{graphviz}
digraph {
  rankdir="BT"
  io[shape=box style=filled]
  models[shape=box style=filled]
  geometry[shape=box style=filled]
  pipeline[shape=box style=filled]
  
  videos -> io
  ephys -> io
  glmhmm -> models
  arhmm -> models
  azimuth -> geometry
  distance -> geometry
  
  models -> pipeline
  io -> pipeline
  geometry -> pipeline
}
```

Here we're specifying that our geometry functions shouldn't know specifically about how the data is loaded, nor how they'll
be combined with the different models that we have. The pipeline-level classes and functions are the only ones that know how 
each of the subtypes of objects work. The graph is intentionally abstract because it's just intended to structure our work.
As we continue to work on the package, our graph becomes a sort of implicit rubber duck test: does my code resemble the graph?
if not, why? does the graph need to be update, or does my code need to be refactored?

The syntax for graphviz is straightforward, the above diagram took ~3 minutes to sketch:


```
digraph {
  rankdir="BT"
  io[shape=box style=filled]
  models[shape=box style=filled]
  geometry[shape=box style=filled]
  pipeline[shape=box style=filled]
  
  videos -> io
  ephys -> io
  glmhmm -> models
  arhmm -> models
  azimuth -> geometry
  distance -> geometry
  
  models -> pipeline
  io -> pipeline
  geometry -> pipeline
}
```

## See Also

- [PyOpenSci Packaging Guide](https://www.pyopensci.org/python-package-guide/)
