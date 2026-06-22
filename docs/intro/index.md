# Introduction

This section covers the foundational concepts you need before writing custom code or building headless applications with Drupal 11.

## Chapters

- **[What Changed in Drupal 11](what-changed.md)** — PHP 8.3+, Symfony 7, OOP hooks, removed modules
- **[Drupal 11 Architecture](architecture.md)** — layered architecture, kernel, subsystems
- **[Modern PHP Standards](modern-php.md)** — typed properties, enums, attributes, readonly
- **[Symfony Foundation](symfony-foundation.md)** — why Drupal is built on Symfony components
- **[Composer-First Development](composer-first.md)** — managing Drupal as a Composer project
- **[Why Headless Drupal](why-headless.md)** — decoupled vs traditional, when to go headless

## Drupal 11 at a glance

```mermaid
graph TB
    subgraph frontend [Frontend Layer]
        NextJS[Next.js / React / Vue]
        Mobile[Mobile App]
    end

    subgraph drupal [Drupal 11 Backend]
        JSONAPI[JSON:API]
        REST[REST API]
        CustomAPI[Custom Controllers]
        subgraph core [Drupal Core]
            Entity[Entity API]
            Plugin[Plugin System]
            Form[Form API]
            Cache[Cache System]
        end
        subgraph symfony [Symfony 7 Components]
            HttpKernel[HttpKernel]
            Routing[Routing]
            DI[DependencyInjection]
            Serializer[Serializer]
        end
    end

    NextJS --> JSONAPI
    Mobile --> REST
    NextJS --> CustomAPI
    JSONAPI --> Entity
    REST --> Entity
    CustomAPI --> Entity
    Entity --> Plugin
    core --> symfony
```
