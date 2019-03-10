# Use MADR-like solution for spike outcomes

* Authors: [Lucas Soriano](@luckysori)
* Date: 2019-03-05

Tracking issue: [#36](https://github.com/coblox/docs/issues/36)

## Context

Currently, we store spike outcomes as comments in the related issues. This works but makes reviewing and giving feedback very inefficient. Moreover, it can make it difficult to look for the reason why a decision was made a later stage.

## Research

### Pros of current system

- Discussions nicely documented on spike issue.
- Stays on GitHub, next to the code.

### Problems with current system

- Hard to identify the outcome of a spike, which could be lost in a comment thread.
- Format of spike outcome is undefined.
- Cumbersome feedback/review process.

### Decision

Use a [MADR](https://github.com/adr/madr)-like approach.

Steps to be followed when doing spikes:

1. Document steps and outcomes of research in Markdown.
2. Submit a PR of the Markdown file to this repository as soon as the spike outcomes are ready to be discussed. Additionally, update the index file accordingly.
3. Refine and discuss during PR review.
4. Merge the PR once consensus has been reached.
