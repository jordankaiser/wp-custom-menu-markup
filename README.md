# WordPress Menu Implementation Standard — The Data Provider Pattern

> **Purpose.** This is a reusable, project-agnostic standard for implementing or
> refactoring navigation menus on **any** WordPress site. Point an agent at this
> file and instruct: *"Implement menus following WORDPRESS-MENU-STANDARD.md,
> adapting it to this site's architecture."* It is both a **specification** of the
> pattern and a **prompt** for carrying it out. It assumes nothing about theme
> structure, build tooling, or naming conventions — Step 0 tells the agent how to
> discover those.

---

## The goal (the three criteria)

Render WordPress menus so that you get **all three** of these simultaneously:

1. **Full control over the markup** — every tag, class, and attribute is authored
   in a template, not dictated by `wp_nav_menu()` or a walker.
2. **All the benefits of a WordPress menu** — context classes
   (`current-menu-item`, `current-menu-ancestor`…), the attributes WordPress
   injects on links, and compatibility with plugins that filter menus.
3. **Strict separation of business logic from display logic** — data preparation
   in one place, markup in another, neither leaking into the other.

### Why not the common approaches

| Approach | Markup control | WP benefits | Logic/display separation |
|---|---|---|---|
| `wp_nav_menu()` (default) | ❌ Core dictates markup | ✅ | ✅ |
| Custom `Walker_Nav_Menu` | ⚠️ Markup lives in PHP walker methods | ✅ | ❌ HTML embedded in logic |
| **Data provider (this standard)** | ✅ Template owns 100% of HTML | ✅ | ✅ |

A walker is the usual escape hatch, but it fails criterion #3: you end up writing
`<li>`/`<ul>`/`<a>` strings inside walker methods, fusing display into business
logic.

### The key insight

`wp_nav_menu()` works in two phases:

1. **Prepare** — load the menu items, apply context classes, run the
   `wp_nav_menu_objects` filter, then run per-item filters
   (`nav_menu_css_class`, `nav_menu_item_id`, `nav_menu_link_attributes`,
   `nav_menu_item_title`, `nav_menu_item_args`).
2. **Render** — a walker turns the prepared items into HTML strings.

**Only the render phase produces markup.** This standard reproduces the *prepare*
phase faithfully in a single provider function, and lets your templates own the
*render* phase. That is what makes "full markup control" coexist with "all the WP
benefits."

---

## Step 0 — Discover the site's architecture (do this first)

Architectures differ across sites. Before writing code, find:

1. **How menus are registered.** Search for `register_nav_menus` /
   `register_nav_menu` to get the list of **theme locations** (e.g. `primary`,
   `footer`). Always target menus by **location**, never by menu title/slug.
2. **Where menus are rendered today (if at all).** Search for `wp_nav_menu`,
   `Walker_Nav_Menu`, `wp_get_nav_menu_items`, and any custom walker class. List
   every call site you find.
3. **Where shared functions live.** Find where the theme/plugin includes its PHP
   helpers (e.g. `functions.php`, an `inc/`/`functions/` loader, a mu-plugin).
   The provider function goes there.
4. **How templates are structured & included.** Note the conventions
   (`get_template_part`, `locate_template`, block templates, a component system,
   etc.) so your rendering templates match house style.
5. **The build/asset pipeline.** Note how CSS/JS are built and any prerequisites
   (Node version via `nvm`, `npm run build`, Composer, etc.) so you can build and
   verify.
6. **Existing markup contract (only if the site already renders these menus).**
   Capture the HTML each menu currently emits and the classes/attributes its CSS
   and JS depend on, so your new markup can reproduce that contract.

Adapt all names, paths, and conventions below to what you find.

---

## Step 1 — Add the provider function

Add **one** reusable function to the site's shared-functions location. It is the
single source of menu data for every nav component. The implementation below is
**portable and works on any standard WordPress site** — copy it as-is.

```php
<?php

/**
 * Build a normalized, fully-hydrated menu tree for a registered nav location.
 *
 * Reproduces every data-preparation step wp_nav_menu() performs — context
 * classes, the wp_nav_menu_objects filter, and the per-item filters
 * (nav_menu_css_class, nav_menu_item_id, nav_menu_link_attributes,
 * nav_menu_item_title) — but emits NO markup. Templates render the HTML.
 *
 * Each returned item is an array:
 *   - id          string   "menu-item-{ID}" (filtered)
 *   - title       string   filtered title
 *   - url         string
 *   - li_classes  string[] classes for the <li> (filtered, incl. menu-item-{ID})
 *   - link_atts   array    href/title/target/rel/aria-current/... (filtered)
 *   - current     bool
 *   - children    array    child items of the same shape (recursive)
 *
 * @param string $location A theme location registered via register_nav_menus().
 * @return array Top-level items, each with a `children` array. Empty if none.
 */
function get_nav_menu_tree( $location ) {
    $locations = get_nav_menu_locations();

    if ( empty( $locations[ $location ] ) ) {
        return array();
    }

    $items = wp_get_nav_menu_items( $locations[ $location ] );

    if ( ! $items ) {
        return array();
    }

    // current-menu-item, current-menu-ancestor, etc. — same as wp_nav_menu().
    _wp_menu_item_classes_by_context( $items );

    // A faithful args object so the core filters below receive what they expect.
    $args = (object) array(
        'theme_location' => $location,
        'menu'           => $locations[ $location ],
        'container'      => false,
        'before'         => '',
        'after'          => '',
        'link_before'    => '',
        'link_after'     => '',
        'depth'          => 0,
    );

    // Allow plugins to filter the item set before output.
    $items = apply_filters( 'wp_nav_menu_objects', $items, $args );

    $prepared = array();

    foreach ( $items as $item ) {
        $depth = 0; // Flat pass; nesting is rebuilt below.
        $args  = apply_filters( 'nav_menu_item_args', $args, $item, $depth );

        // --- Classes (mirrors Walker_Nav_Menu::start_el) ---
        $classes   = empty( $item->classes ) ? array() : (array) $item->classes;
        $classes[] = 'menu-item-' . $item->ID;
        $li_classes = apply_filters( 'nav_menu_css_class', array_filter( $classes ), $item, $args, $depth );

        // --- Id ---
        $id = apply_filters( 'nav_menu_item_id', 'menu-item-' . $item->ID, $item, $args, $depth );

        // --- Link attributes (verbatim from core) ---
        $atts                 = array();
        $atts['title']        = ! empty( $item->attr_title ) ? $item->attr_title : '';
        $atts['target']       = ! empty( $item->target ) ? $item->target : '';
        if ( '_blank' === $item->target && empty( $item->xfn ) ) {
            $atts['rel'] = 'noopener';
        } else {
            $atts['rel'] = $item->xfn;
        }
        $atts['href']         = ! empty( $item->url ) ? $item->url : '';
        $atts['aria-current'] = $item->current ? 'page' : '';
        $atts = apply_filters( 'nav_menu_link_attributes', $atts, $item, $args, $depth );

        // Drop empty attributes so templates don't emit `target=""` etc.
        $atts = array_filter( $atts, static function ( $value ) {
            return '' !== $value && null !== $value && false !== $value;
        } );

        // --- Title ---
        $title = apply_filters( 'the_title', $item->title, $item->ID );
        $title = apply_filters( 'nav_menu_item_title', $title, $item, $args, $depth );

        $prepared[ $item->ID ] = array(
            'ID'         => (int) $item->ID,
            'parent'     => (int) $item->menu_item_parent,
            'id'         => $id,
            'title'      => $title,
            'url'        => $item->url,
            'li_classes' => array_values( $li_classes ),
            'link_atts'  => $atts,
            'current'    => (bool) $item->current,
            'children'   => array(),
        );
    }

    // --- Assemble the tree by parent id (supports unlimited depth) ---
    $top_level = array();

    foreach ( $prepared as $id => $item ) {
        if ( $item['parent'] && isset( $prepared[ $item['parent'] ] ) ) {
            $prepared[ $item['parent'] ]['children'][] = &$prepared[ $id ];
        } else {
            $top_level[] = &$prepared[ $id ];
        }
    }

    return $top_level;
}

/**
 * Render a `link_atts` array (from get_nav_menu_tree) into an escaped attribute
 * string for an <a> tag. Each value is escaped with esc_attr().
 *
 * Usage:  <a <?php echo nav_render_atts( $item['link_atts'] ); ?>>
 *
 * @param array $atts Associative array of attribute => value.
 * @return string Leading-space-prefixed attribute string, or '' if empty.
 */
function nav_render_atts( $atts ) {
    $out = '';

    foreach ( $atts as $name => $value ) {
        $out .= ' ' . $name . '="' . esc_attr( $value ) . '"';
    }

    return $out;
}
```

> **Do not "simplify" the `$atts` block or the `_blank`/`xfn` rule** — they are
> copied verbatim from `Walker_Nav_Menu::start_el()`. That fidelity is the entire
> point: it reproduces exactly the attributes WordPress (and plugins) add.
>
> **Naming/namespacing:** prefix the function names to match the project
> (`mytheme_get_nav_menu_tree()`, a static class method, etc.) to avoid global
> collisions. Keep the behavior identical.

---

## Step 2 — Render in templates

Templates **only loop and echo**. No WordPress function calls, no attribute logic.
The `href` lives inside `link_atts`, so a link is `<a<?php echo
nav_render_atts( $item['link_atts'] ); ?>>`.

Generic recursive-capable example:

```php
<?php $items = get_nav_menu_tree( 'primary' ); // adapt location ?>

<?php if ( $items ) : ?>
<ul class="menu">
  <?php foreach ( $items as $item ) : ?>
    <li id="<?php echo esc_attr( $item['id'] ); ?>"
        class="<?php echo esc_attr( implode( ' ', $item['li_classes'] ) ); ?>">

      <a<?php echo nav_render_atts( $item['link_atts'] ); ?>>
        <?php echo esc_html( $item['title'] ); ?>
      </a>

      <?php if ( ! empty( $item['children'] ) ) : ?>
        <ul class="sub-menu">
          <?php foreach ( $item['children'] as $child ) : ?>
            <li id="<?php echo esc_attr( $child['id'] ); ?>"
                class="<?php echo esc_attr( implode( ' ', $child['li_classes'] ) ); ?>">
              <a<?php echo nav_render_atts( $child['link_atts'] ); ?>>
                <?php echo esc_html( $child['title'] ); ?>
              </a>
            </li>
          <?php endforeach; ?>
        </ul>
      <?php endif; ?>

    </li>
  <?php endforeach; ?>
</ul>
<?php endif; ?>
```

This is the *baseline* shape. Real components keep their own markup — buttons,
accordions, mega-menus, ARIA wiring — and simply source data from
`get_nav_menu_tree()` and echo `link_atts`. For arbitrary depth, extract the
`<li>` rendering into a small recursive template partial that calls itself on
`$item['children']`.

---

## Verification checklist

- [ ] All `register_nav_menus` locations and all `wp_nav_menu`/walker call sites
      were discovered and accounted for.
- [ ] Provider function added (prefixed per project) and templates source from it.
- [ ] Build succeeds (with any project-specific prerequisites, e.g. correct Node
      version).
- [ ] `current-menu-item` class and `aria-current="page"` appear on the active
      item; ancestor classes on its parents.
- [ ] External links retain `target` and the correct `rel` (`noopener` / `xfn`).
- [ ] Multi-level menus nest correctly.
- [ ] Any menu-related plugin still works (its filters run inside the provider).

---

## One-paragraph summary (for prompting)

> Implement WordPress menus using a **data provider** function,
> `get_nav_menu_tree( $location )`, that reproduces everything `wp_nav_menu()`
> does *before* it renders — context classes, the `wp_nav_menu_objects` filter,
> and the per-item filters `nav_menu_css_class`, `nav_menu_item_id`,
> `nav_menu_link_attributes`, and `nav_menu_item_title` — and returns a
> normalized, markup-free nested array. Templates iterate that array and own 100%
> of the HTML, echoing prepared link attributes via a small `nav_render_atts()`
> helper. This gives full markup control, full WordPress/plugin compatibility, and
> a clean split between business logic (the provider) and display logic (the
> templates) — without any custom walker.
