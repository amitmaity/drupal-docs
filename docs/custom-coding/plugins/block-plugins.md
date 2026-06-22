# Block Plugins

Block plugins render reusable chunks of content placed via the Block layout UI or programmatically.

## Block plugin — `src/Plugin/Block/RecentItemsBlock.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Plugin\Block;

use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Lists recent nodes of a selected type.
 */
#[Block(
  id: 'example_module_recent_items',
  admin_label: new TranslatableMarkup('Recent items'),
  category: new TranslatableMarkup('Custom'),
)]
final class RecentItemsBlock extends BlockBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('entity_type.manager'),
    );
  }

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return [
      'bundle' => 'article',
      'count' => 5,
    ] + parent::defaultConfiguration();
  }

  /**
   * {@inheritdoc}
   */
  public function blockForm(array $form, FormStateInterface $form_state): array {
    $form['bundle'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Content type machine name'),
      '#default_value' => $this->configuration['bundle'],
      '#required' => TRUE,
    ];

    $form['count'] = [
      '#type' => 'number',
      '#title' => $this->t('Number of items'),
      '#default_value' => $this->configuration['count'],
      '#min' => 1,
      '#max' => 20,
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function blockSubmit(array $form, FormStateInterface $form_state): void {
    $this->configuration['bundle'] = $form_state->getValue('bundle');
    $this->configuration['count'] = (int) $form_state->getValue('count');
  }

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    $bundle = (string) $this->configuration['bundle'];
    $count = (int) $this->configuration['count'];

    $storage = $this->entityTypeManager->getStorage('node');
    $nids = $storage->getQuery()
      ->accessCheck(TRUE)
      ->condition('type', $bundle)
      ->condition('status', 1)
      ->sort('created', 'DESC')
      ->range(0, $count)
      ->execute();

    $items = [];
  $nodes = $storage->loadMultiple($nids);
    foreach ($nodes as $node) {
      $items[] = [
        '#type' => 'link',
        '#title' => $node->label(),
        '#url' => $node->toUrl(),
      ];
    }

    return [
      '#theme' => 'item_list',
      '#items' => $items,
      '#empty' => $this->t('No items found.'),
      '#cache' => [
        'tags' => ['node_list:' . $bundle],
        'contexts' => ['user.permissions'],
      ],
    ];
  }

}
```

---

## Place block programmatically

```php
<?php

declare(strict_types=1);

use Drupal\block\Entity\Block;

$block = Block::create([
  'id' => 'example_module_recent_items_sidebar',
  'plugin' => 'example_module_recent_items',
  'region' => 'sidebar_first',
  'theme' => \Drupal::config('system.theme')->get('default'),
  'settings' => [
    'label' => 'Recent articles',
    'label_display' => 'visible',
    'bundle' => 'article',
    'count' => 5,
  ],
  'visibility' => [],
  'weight' => 0,
]);
$block->save();
```

---

## Render block in a controller

```php
<?php

declare(strict_types=1);

namespace Drupal\example_module\Controller;

use Drupal\Core\Block\BlockManagerInterface;
use Drupal\Core\Controller\ControllerBase;
use Symfony\Component\DependencyInjection\ContainerInterface;

final class BlockDemoController extends ControllerBase {

  public function __construct(
    private readonly BlockManagerInterface $blockManager,
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container): static {
    return new static($container->get('plugin.manager.block'));
  }

  public function demo(): array {
    $plugin = $this->blockManager->createInstance('example_module_recent_items', [
      'bundle' => 'article',
      'count' => 3,
    ]);

    return $plugin->build();
  }

}
```

---

## `access` method

Restrict who sees the block:

```php
/**
 * {@inheritdoc}
 */
protected function blockAccess(AccountInterface $account): AccessResult {
  return AccessResult::allowedIfHasPermission($account, 'access content')
    ->addCacheContexts(['user.permissions']);
}
```

!!! tip "Place via UI"
    After enabling the module: **Structure → Block layout → Place block** → search for "Recent items".

!!! warning "Cache tags"
    Always include relevant entity list cache tags so content updates invalidate the block automatically.
