# Getting Started

This section walks you through installing Drupal 11, understanding the project layout, and using essential developer tools.

## Chapters

- **[Fresh Installation](installation.md)** — DDEV + Composer setup
- **[Composer Workflows](composer.md)** — require, update, patches, scaffold
- **[Project Layout](project-layout.md)** — directory structure explained
- **[Drush Basics](drush.md)** — command-line site management

## Minimum requirements

| Requirement | Version |
|-------------|---------|
| PHP | 8.3+ |
| Composer | 2.x |
| Database | MySQL 8.0+ / MariaDB 10.6+ / PostgreSQL 16+ |
| Web server | Apache 2.4+ / Nginx 1.24+ |
| Memory | 256 MB PHP memory limit (512 MB recommended) |

## Recommended tools

```bash
# DDEV for local development
brew install ddev/ddev/ddev

# Drush for CLI management
composer require drush/drush

# PHP CodeSniffer for standards
composer require --dev drupal/coder
# Standards are registered automatically; confirm Drupal and DrupalPractice are listed
vendor/bin/phpcs -i
```
