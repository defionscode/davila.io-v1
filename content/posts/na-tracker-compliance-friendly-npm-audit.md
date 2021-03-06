---
title: "npm-audit-tracker: An Open Source, Compliance-Friendly Utility for npm audit"
date: 2019-01-28T11:31:36-05:00
showDate: true
draft: true
tags: ["node","javascript","serverless","compliance","npm"]
---

![Screenshot from a sample GitHub repo showing the types of issues opened up by npm-audit-tracker](/images/na-tracker.webp)

**TL;DR** We've [open-sourced yet another utility that can help others][tracker]. It automates tracking when `npm audit` detects a vulnerability and notifies stakholders by opening detailed GitHub issues while also automatically closes GitHub issues when the vulnerability is no longer present. It does this at every time interval you desire and anytime there is a push to master (or whatever branch you define). It is implemented in and runs entirely on AWS Lambda.

> With `npm@6` you get a wonderful utility for dependency vulnerability detection via `npm audit`. A detailed description is provided by [npm in their documentation][audit-docs]. This utility is wonderful by itself and has proven itself valuable to the JavaScript community at-large.

## Where

The tool is free and open and available on [GitHub][tracker]

## Why

For many compliance bodies, like FedRAMP for example, there exists an explicit requirement to keep track of _when_ a vulnerability was detected and _when_ it was remediated. This is the one place were `npm audit` alone does not suffice. Of course, you could do some clever checking with git history and the sort but that is a non-trivial path and subject to user error.

My team is developing a full-stack JavaScript web application and one of our goals to achieve compliance with FedRAMP. However, quite a bit of work around compliance is hard (and seemingly impossible) to automate; we try still do it. With dependency vulnerability management we felt this was one thing we could automate well and with minimal effort.

## How

When contemplating how we could track the two necessary data points:

1. Time of detection; and
2. Time of remediation

I considered a few paths and decided that the easiest path forward would be to leverage what has already been built and simply tie the pieces together.

### Vulnerability Detection

It's not immediatly obvious but if you do `npm audit --json` you get a very nice machine-readable version of the human-friendly version outputted by `npm audit`. This, of course, was to be the way we would find things.

### Notifying on finding

An important ability is being able to announce findings in a way that was open and obvious to project collaborators. There was no need to devise an email utility or anything like that, opening a GitHub issue via the GitHub API would work just as well (bonus: emailing 'watchers' is included). Additionally we could simply `@<SomeStakeholder>` in the body of the GitHub issue to be _really_ sure that the right people would get pinged. Additionally, this would satisfy the data point of "Time of Detection". The [GitHub API][list-issues] includes the following data when listing issues in a repo:

```json
...
    },
    "closed_at": null,
    "created_at": "2011-04-22T13:33:48Z",
    "updated_at": "2011-04-22T13:33:48Z"
  }
]
```

We can leverage `created_at` as the data point for when the vulnerability was detected.

Here's an example screenshot of an GitHub issue generated by this tool:

![Screenshot from a sample GitHub repo issue showing the specific details of a found vulnerability](/images/example-issue-detail.webp)

### Triggering a scan

For scanning I realized we needed two methods. We need a scan anytime that master changes (really just `package.json` but it's easier just to do it on every push to master), and we need a scan at some regular interval for when a current dependency is found to be vulnerable.

This is handled easily via GitHub Webhooks (for push events) and through a [scheduled Lambda][lambda-sched] (we set ours for 12 hour intervals).

### Detecting remediation

To do this we simply piggybacked on the same logic. When an event is triggered (interval or push) we have the Lambda function re-run `npm audit --json` and if an open GitHub issue exists, but is not detected in the output then it will automatically close the issue. Since the GitHub API exposes a `closed_at` property we can now leverage that data to provide a remediation date for a given vulnerability.

## Roadmap

This is a very early-stage tool and we have plans to add other things we think would be useful and relevant. This list isn't set in stone but all the items are very probable:

* Support for correlation with GitHub vulnerability data
* Support for other languages (starting with Python)
* Convert into a "real" GitHub Application
* Create some basic reporting utilities that scrape the GitHub API.

## Thank You

If you've made it this far, thank you for reading. We faced a particular challenge, and I think many of you face the same struggle. [I hope that this can help make your life a little bit easier][tracker].

End.

[lambda-sched]:https://docs.aws.amazon.com/lambda/latest/dg/tutorial-scheduled-events-schedule-expressions.html
[list-issues]: https://developer.github.com/v3/issues/#list-issues-for-a-repository
[audit-docs]: https://docs.npmjs.com/auditing-package-dependencies-for-security-vulnerabilities
[tracker]:https://github.com/MindPointGroup/npm-audit-tracker