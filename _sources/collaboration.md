# Collaboration

## Continuous Integration

Why CI helps us do that! 

Example of how CI works in practice looking at PVP pull requests!

Example of travis configuration from autopilot!

This is part of a broader set of commit-triggered things, like this page is built using a github action!

## Making git your friend

* **COMMIT OFTEN** - if you don't want to "mess up" the main branch, just make another branch! It should *never* be the case
  that code only exists on your computer.

How to use git with other people! Example of branching and merging!

Corrolary - don't comment code out. use git!!!!!

## Separating Configuration from Code

The major reason for hardcoding things is to make up for implicit structure. We make lists of directories
and files because we don't have a well defined notion of "project directories" and "datasets" that would
let us find them automatically. 

Example of configuration using pydantic.

## Explicit Metadata

Metadata can end up drifting into filenames. Instead what we can do is make that explicit...

