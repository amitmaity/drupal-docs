# Deploying Drupal

While this guide focuses on headless Drupal development, here is a concise overview of deploying Drupal 11 backends to production.

## Hosting options

| Platform | Best for | Headless support |
|----------|----------|------------------|
| [Platform.sh](https://platform.sh/) | Enterprise, multi-environment | Excellent |
| [Acquia Cloud](https://www.acquia.com/) | Enterprise Drupal | Excellent |
| [Pantheon](https://pantheon.io/) | Agencies, mid-market | Good |
| [Lagoon](https://lagoon.sh/) | Open-source, Kubernetes | Excellent |
| [AWS / GCP / Azure](https://aws.amazon.com/) | Full control, custom infra | DIY |
| [DDEV](https://ddev.com/) | Local development only | N/A |

## Production checklist

```bash
# 1. Install without dev dependencies
composer install --no-dev --optimize-autoloader

# 2. Run database updates, import configuration, rebuild caches
#    and run deploy hooks, in the correct order
#    (updatedb → config:import → cache:rebuild → deploy:hook)
drush deploy -y

# 3. Verify file permissions
chmod 444 web/sites/default/settings.php
chmod -R 755 web/sites/default/files
```

## Headless-specific configuration

### CORS for production

```yaml
# services.yml (production)
parameters:
  cors.config:
    enabled: true
    allowedHeaders: ['Accept', 'Authorization', 'Content-Type', 'X-CSRF-Token']
    allowedMethods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS']
    allowedOrigins: ['https://www.myfrontend.com']
    supportsCredentials: true
    maxAge: 3600
```

### Trusted host patterns

```php
// settings.php
$settings['trusted_host_patterns'] = [
  '^api\.myproductionsite\.com$',
];
```

### Reverse proxy (behind CDN/load balancer)

```php
// settings.php
$settings['reverse_proxy'] = TRUE;
// List only your own load balancer / CDN IPs or CIDR ranges.
// Never use $_SERVER['REMOTE_ADDR'] here: it trusts every client, so anyone
// can spoof their IP with an X-Forwarded-For header.
$settings['reverse_proxy_addresses'] = [
  '10.0.0.10',
  '10.0.1.0/24',
];
```

## Environment variables

Use environment variables for secrets:

```php
// settings.php
$databases['default']['default'] = [
  'database' => getenv('DB_NAME') ?: 'drupal',
  'username' => getenv('DB_USER') ?: 'drupal',
  'password' => getenv('DB_PASS') ?: '',
  'host' => getenv('DB_HOST') ?: 'localhost',
  'driver' => 'mysql',
];

$config['simple_oauth.settings']['private_key'] = getenv('OAUTH_PRIVATE_KEY_PATH');
$config['simple_oauth.settings']['public_key'] = getenv('OAUTH_PUBLIC_KEY_PATH');
```

## Deployment pipeline example

```mermaid
graph LR
    Dev[Developer Push] --> CI[GitHub Actions CI]
    CI --> Test[PHPUnit + PHPStan]
    Test --> Build[Composer Install]
    Build --> Deploy[Deploy to Platform]
    Deploy --> Updatedb[drush updatedb]
    Updatedb --> ConfigImport[drush config:import]
    ConfigImport --> CacheClear[drush cr + deploy hooks]
    CacheClear --> Live[Production API Live]
```

## API-only production tips

!!! tip "Disable unused modules"
    On API-only sites, disable modules you don't need (Views UI, contextual, etc.) to reduce attack surface.

!!! tip "Enable aggregation"
    ```bash
    drush config:set system.performance css.preprocess 1 -y
    drush config:set system.performance js.preprocess 1 -y
    ```

!!! tip "Set up monitoring"
    Monitor JSON:API response times, queue depths, and cron execution. Use New Relic, Datadog, or Platform.sh metrics.

!!! warning "Never commit secrets"
    Keep `settings.php` secrets out of version control. Use environment variables or platform secrets management.

## Further reading

- [Drupal.org deployment guide](https://www.drupal.org/docs/getting-started/installing-drupal)
- [Headless architecture](../headless/architecture.md)
- [Authentication](../headless/auth/index.md)
- [Cache invalidation](../headless/cache-invalidation.md)
