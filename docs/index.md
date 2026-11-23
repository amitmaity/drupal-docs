# Drupal 11 Headless & Custom Coding Guide

Welcome to the definitive developer guide for **Drupal 11** with a focus on **headless/decoupled architecture** and **custom module development**.

This guide teaches you how Drupal works under the hood, how every Symfony component fits in, and how to build production-grade APIs and custom modules with real, copy-pasteable code.

## Who this guide is for

- PHP developers new to Drupal who want to understand the architecture
- Drupal developers transitioning to headless/decoupled projects
- Frontend developers consuming Drupal APIs (Next.js, React, Vue)
- Teams evaluating Drupal 11 for a decoupled CMS strategy
- Any PHP developer moving a codebase, or their own habits, from PHP 5 or 7 to PHP 8

## Prerequisites

- PHP 8.3 or later
- Composer 2.x
- Basic familiarity with OOP PHP and REST APIs
- [DDEV](https://ddev.com/) or Docker for local development (recommended)

## What you'll learn

| Section | Topics |
|---------|--------|
| [Introduction](intro/index.md) | What changed in D11, architecture, modern PHP, why headless |
| [PHP 5 to 8](php/index.md) | Every language change from PHP 5.6 to 8.6, breaking changes, upgrade strategy (framework-neutral) |
| [Getting Started](getting-started/index.md) | Fresh install, Composer workflows, Drush |
| [Drupal Internals](internals/index.md) | Bootstrap, request lifecycle, core systems |
| [Dependency Injection](dependency-injection/index.md) | Service container, tagged services, decoration |
| [Symfony Components](symfony/index.md) | Every Symfony component Drupal uses |
| [Custom Modules](custom-coding/index.md) | Forms, plugins, entities, hooks, batch/queue/cron |
| [Headless Drupal](headless/index.md) | REST, JSON:API, auth, Next.js integration |
| [Caching](caching/index.md) | Render cache, tags, contexts, invalidation |

## Quick start

Install Drupal 11 with Composer and DDEV:

```bash
mkdir my-drupal-site && cd my-drupal-site
ddev config --project-type=drupal11 --docroot=web
ddev start
ddev composer create "drupal/recommended-project:^11" --no-install
ddev composer install
ddev composer require drush/drush
ddev drush site:install --account-name=admin --account-pass=admin -y
```

Enable JSON:API for headless development:

```bash
ddev drush en jsonapi -y
curl -s http://my-drupal-site.ddev.site/jsonapi | jq .
```

## How this guide is organized

Every page includes:

1. **Concept explanation** — what it is and why it matters
2. **Drupal wiring** — how core uses it
3. **Code example** — working PHP, YAML, or JavaScript you can adapt

!!! tip "Drupal 11.1+ OOP Hooks"
    Starting in Drupal 11.1, hooks can be implemented as class methods with the `#[Hook]` attribute. This guide shows both OOP and legacy procedural styles.

## Live documentation

This site is deployed at **[amitmaity.github.io/drupal-docs](https://amitmaity.github.io/drupal-docs/)**.
