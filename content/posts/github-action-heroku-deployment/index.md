---
title: GitHub Action reporting (Heroku) deployment status
slug: github-action-heroku-deployment
date: 2021-02-04T13:18:48Z
summary: How to create a GitHub Action that simply fails when status of a
  deployment fails (e.g., deployment reported by Heroku from building a Review
  App).
tags:
  - GitHub Actions
  - Heroku
categories:
  - DevOps
cover:
  image: cover.png
  alt: GitHub Actions + Heroku
  relative: true
ShowToc: true
TocOpen: true
---

I use [Heroku](https://www.heroku.com/) and its [Review Apps](https://devcenter.heroku.com/articles/github-integration-review-apps) functionality to deploy every Pull Request of a web application project.
This way any changes can be reviewed before merging and deploying to production (or staging) version.

Heroku automatically creates a [deployment](https://docs.github.com/en/rest/reference/repos#deployments) in my GitHub repository for each Review App and updates its state whenever the deployment succeeds or fails.
However, when the deployment fails, the corresponding Pull Request can still be merged!
{{< figure src="deployment.png"
    link="deployment.png" rel="true"
    alt="Deployment Screenshot" >}}

In this blog post, we will see how to create a [GitHub Action](https://github.com/features/actions) that automatically translates a deployment status into a [status check](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/about-status-checks) for the corresponding PR, so you can for example forbid it from being merging using [required status checks](https://docs.github.com/en/github/administering-a-repository/about-required-status-checks).
{{< figure src="status-checks.png"
    link="status-checks.png" rel="true"
    alt="Status Checks Screenshot" >}}

## Existing solutions

There are already some GitHub Actions that try to solve this problem.
For example, [Heroku Review App Deployment Status](https://github.com/marketplace/actions/heroku-review-app-deployment-status).
However, they use polling HTTP requests to wait until the deployment succeeds or fails.
And that costs you [Action minutes](https://docs.github.com/en/github/setting-up-and-managing-billing-and-payments-on-github/about-billing-for-github-actions).
My complex web project takes approx. 10 minutes to build in Heroku, so this solution would be unfeasible.

The aforementioned Action's authors propose a [solution](https://blog.niteo.co/staging-like-its-2020/) where you need to change your Heroku deployment script so that it creates a status check in your GitHub repository when it finishes.
However, that seems overly complicated and I propose alternative easier approach here.

## My solution

We can use the [`deployment_status` trigger](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#deployment_status) to execute a GitHub Action only *after* the Heroku deployment finishes.
Here is the code (place it for example into `.github/workflows/heroku.yml`):

```yml
name: Check Deployment Status

on: deployment_status

jobs:
  deployment-status:

    runs-on: ubuntu-latest

    # Continue only if some definitive status has been reported.
    if: github.event.deployment_status.state != 'pending'

    steps:
    # Forward deployment's status to the deployed commit.
    - uses: octokit/request-action@v2.0.23
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        route: POST /repos/:repository/statuses/${{ github.event.deployment.sha }}
        repository: ${{ github.repository }}
        state: ${{ github.event.deployment_status.state }}
        target_url: https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}
        description: >
          ${{ format('Heroku deployment is {0}', github.event.deployment_status.state) }}
        context: deployment-status
```

It simply forwards the status of a Heroku deployment (`success` or `failure`) into status of the Pull Request's last commit.
And it only runs 20 seconds on average, that is a huge improvement over 10 minutes, right?
{{< figure src="deployment-status-action.png"
    link="deployment-status-action.png" rel="true"
    alt="GitHub Action Run Screenshot" >}}

### Bonus: Health check

As you might have noticed from the screenshot in the introduction, I also use separate health-check step to verify that the web application actually runs successfully after deployment.
Its code is below (you can simply append it to the same file, e.g., `.github/workflows/heroku.yml`):

```yml
  health-check:

    runs-on: ubuntu-latest

    # Run health check only if deployment succeeds.
    if: github.event.deployment_status.state == 'success'

    # Check that the deployed app returns successful HTTP response.
    steps:
    - id: health_check
      uses: jtalk/url-health-check-action@v1.3
      with:
        url: ${{ github.event.deployment.payload.web_url }}
        follow-redirect: true
        max-attempts: 4
        retry-delay: 30s
      continue-on-error: true
    # Set appropriate status to the deployed commit.
    - uses: octokit/request-action@v2.0.23
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        route: POST /repos/:repository/statuses/${{ github.event.deployment.sha }}
        repository: ${{ github.repository }}
        state: ${{ steps.health_check.outcome }}
        target_url: https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}
        description: >
          ${{ format('Site health is {0}', steps.health_check.outcome) }}
        context: health-check
```

## Limitations

Note that the presented solution is not limited to Heroku deployments, it should work with any service that publishes deployment statuses to GitHub.
However, the solution has one limitation (that I am aware of) which I will try to address in some future post.
Namely, commit hashes are not checked, so if for example deployment succeeds but [Heroku Release Phase](https://devcenter.heroku.com/articles/release-phase) fails, health check succeeds anyway (with previously deployed version of the branch if it exists).

Let me know what you think and feel free to ask questions about this post in
dedicated [thread on
Reddit](https://www.reddit.com/user/jjones_cz/comments/lcfihc/github_action_reporting_heroku_deployment_status/).
