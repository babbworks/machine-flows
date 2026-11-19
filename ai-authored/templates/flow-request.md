---
mach: 1                      # schema version (plan 08 §3.6)
type: flow
title: Short name for this run
flow: research-to-plan       # a recipe in autoflows/mach/ or scripts/mach.d/flows/ (research-to-plan | issue-to-pr | morning | ...)
input: ""                    # what the flow starts from: a question, a tracker issue id, a path
machine: any                 # a machine name from [machines] in the mach config | any
output: ""                   # optional; the recipe sets a default
due: ""                      # YYYY-MM-DD, optional
approval: before-run         # none | before-run | before-apply (the recipe's own gates always apply)
tags: []
---
Anything the flow should know that the input doesn't say: limits, preferences, what to skip.

## Context
- (links, paths, ticket ids)
