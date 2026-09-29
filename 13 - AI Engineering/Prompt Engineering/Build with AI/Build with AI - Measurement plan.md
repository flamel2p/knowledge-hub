---
title: "Build with AI - Measurement plan"
aliases: ["Measurement plan prompt"]
type: reference
domain: ai-engineering
tags: [domain/ai-engineering, type/reference, topic/prompt-engineering, series/build-with-ai]
status: complete
created: 2026-09-29
updated: 2026-09-29
parent: "[[Build with AI]]"
---

> [!info] Part of the [[Build with AI]] prompt playbook series · see [[Prompt Engineering]]

## Step 1: Identify key presentation metrics

```text
I work at an [your company type, e.g. apparel company]. I'm preparing for an [type of meeting, e.g. executive review of website performance]. Our goal is to [primary business goal, e.g. increase revenue in Q3]. What are the top 5 metrics I should consider for my presentation?
```

Review the suggested metrics and pick the ones that map directly to the stated goal. You will use one of them in the next step.


## Step 2: Anticipate leadership questions

```text
What questions might our leadership team ask about [one of the suggested metrics from the previous step]?
```

Review the potential questions. As a next step, ask where to find the data that answers them.


## Optional: Export data from a specific source

If you know where the data lives but not how to export it, ask for step-by-step instructions.

```text
How do I export [data, e.g. website traffic data] from [data source, e.g. Google Analytics] so I can analyze it in [destination, e.g. a Google Sheet]?
```

> [!warning] Verify export steps
> UI paths for tools like Google Analytics change often and the model may describe an outdated menu. Cross-check against the vendor's current docs before relying on the answer.
