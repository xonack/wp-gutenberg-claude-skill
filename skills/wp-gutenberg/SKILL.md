---
name: wp-gutenberg
description: Complete Gutenberg block development reference covering registration, edit/save components, dynamic blocks, InnerBlocks, variations, patterns, supports, build pipeline, transforms, testing, and performance optimization.
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
---

# Gutenberg Block Development

## 1. Block Registration

`block.json` is the canonical registration method since WordPress 5.8+. It enables automatic asset enqueue, server-side discovery, and block directory compatibility.

### block.json Required Fields

```json
{
  "apiVersion": 3,
  "$schema": "https://schemas.wp.org/trunk/block.json",
  "name": "zentratec/hero-banner",
  "version": "1.0.0",
  "title": "Hero Banner",
  "category": "design",
  "icon": "cover-image",
  "description": "A full-width hero banner with heading, text, and CTA.",
  "textdomain": "zentratec",
  "editorScript": "file:./index.js",
  "editorStyle": "file:./index.css",
  "style": "file:./style-index.css",
  "viewScript": "file:./view.js"
}
```

### block.json Optional Fields

```json
{
  "attributes": {
    "heading": { "type": "string", "default": "" },
    "mediaId": { "type": "number" },
    "mediaUrl": { "type": "string", "source": "attribute", "selector": "img", "attribute": "src" }
  },
  "supports": { "align": ["wide", "full"], "color": { "background": true, "text": true } },
  "styles": [
    { "name": "default", "label": "Default", "isDefault": true },
    { "name": "dark", "label": "Dark Overlay" }
  ],
  "variations": [],
  "example": {
    "attributes": { "heading": "Welcome to Zentratec" }
  },
  "parent": ["zentratec/section"],
  "ancestor": ["core/group"],
  "keywords": ["banner", "hero", "cta"]
}
```

### PHP Registration

```php
// plugin.php or functions.php
function zentratec_register_blocks(): void {
    register_block_type( __DIR__ . '/build/hero-banner' );
}
add_action( 'init', 'zentratec_register_blocks' );
```

Register multiple blocks from a shared build directory:

```php
function zentratec_register_all_blocks(): void {
    $blocks = glob( __DIR__ . '/build/blocks/*/block.json' );
    foreach ( $blocks as $block_json ) {
        register_block_type( dirname( $block_json ) );
    }
}
add_action( 'init', 'zentratec_register_all_blocks' );
```

---

## 2. Block Edit Component

The Edit component renders in the editor. `useBlockProps()` is mandatory -- it provides block wrapper attributes (id, className, data-attributes).

```jsx
import { useBlockProps, InspectorControls, BlockControls, RichText, MediaUpload, MediaUploadCheck } from '@wordpress/block-editor';
import { PanelBody, ToolbarGroup, ToolbarButton, RangeControl, ToggleControl, SelectControl } from '@wordpress/components';
import { __ } from '@wordpress/i18n';

export default function Edit( { attributes, setAttributes } ) {
    const { heading, mediaUrl, mediaId, overlayOpacity, showCta } = attributes;
    const blockProps = useBlockProps( { className: 'zentratec-hero' } );

    return (
        <>
            <BlockControls>
                <ToolbarGroup>
                    <MediaUploadCheck>
                        <MediaUpload
                            onSelect={ ( media ) => setAttributes( { mediaId: media.id, mediaUrl: media.url } ) }
                            allowedTypes={ [ 'image' ] }
                            value={ mediaId }
                            render={ ( { open } ) => (
                                <ToolbarButton onClick={ open } icon="format-image" label={ __( 'Edit Image', 'zentratec' ) } />
                            ) }
                        />
                    </MediaUploadCheck>
                </ToolbarGroup>
            </BlockControls>

            <InspectorControls>
                <PanelBody title={ __( 'Hero Settings', 'zentratec' ) }>
                    <RangeControl
                        label={ __( 'Overlay Opacity', 'zentratec' ) }
                        value={ overlayOpacity }
                        onChange={ ( val ) => setAttributes( { overlayOpacity: val } ) }
                        min={ 0 } max={ 100 } step={ 5 }
                    />
                    <ToggleControl
                        label={ __( 'Show CTA Button', 'zentratec' ) }
                        checked={ showCta }
                        onChange={ ( val ) => setAttributes( { showCta: val } ) }
                    />
                </PanelBody>
            </InspectorControls>

            <div { ...blockProps }>
                { mediaUrl && <img src={ mediaUrl } alt="" /> }
                <RichText
                    tagName="h1"
                    value={ heading }
                    onChange={ ( val ) => setAttributes( { heading: val } ) }
                    placeholder={ __( 'Enter heading...', 'zentratec' ) }
                />
            </div>
        </>
    );
}
```

---

## 3. Block Save Component

The Save component outputs static HTML stored in the database. `useBlockProps.save()` mirrors the editor wrapper.

### Static Save

```jsx
import { useBlockProps, RichText } from '@wordpress/block-editor';

export default function Save( { attributes } ) {
    const { heading, mediaUrl } = attributes;
    const blockProps = useBlockProps.save( { className: 'zentratec-hero' } );

    return (
        <div { ...blockProps }>
            { mediaUrl && <img src={ mediaUrl } alt="" /> }
            <RichText.Content tagName="h1" value={ heading } />
        </div>
    );
}
```

### Dynamic Save (return null)

When server-side rendering handles output, save returns `null`. This avoids block validation errors when the markup changes.

```jsx
export default function Save() {
    return null;
}
```

**When to use static vs dynamic:**
- **Static**: Simple blocks with fixed markup. Faster rendering, no PHP overhead per page load.
- **Dynamic**: Blocks that query posts, display user-specific data, or have markup that evolves across plugin versions.

---

## 4. Dynamic Blocks

### render_callback in PHP

```php
register_block_type( __DIR__ . '/build/recent-posts', [
    'render_callback' => 'zentratec_render_recent_posts',
] );

function zentratec_render_recent_posts( array $attributes, string $content, WP_Block $block ): string {
    $count = $attributes['count'] ?? 3;
    $posts = get_posts( [
        'numberposts' => $count,
        'post_status' => 'publish',
    ] );

    if ( empty( $posts ) ) {
        return '<p>' . esc_html__( 'No posts found.', 'zentratec' ) . '</p>';
    }

    $wrapper = get_block_wrapper_attributes( [ 'class' => 'zentratec-recent-posts' ] );
    $output  = "<div {$wrapper}><ul>";

    foreach ( $posts as $post ) {
        $output .= sprintf(
            '<li><a href="%s">%s</a></li>',
            esc_url( get_permalink( $post ) ),
            esc_html( $post->post_title )
        );
    }

    return $output . '</ul></div>';
}
```

### render in block.json (WP 6.1+)

```json
{
  "render": "file:./render.php"
}
```

```php
<?php
// render.php -- $attributes, $content, and $block are available automatically
$wrapper = get_block_wrapper_attributes();
$heading = esc_html( $attributes['heading'] ?? '' );
?>
<div <?php echo $wrapper; ?>>
    <h2><?php echo $heading; ?></h2>
    <?php echo $content; ?>
</div>
```

---

## 5. InnerBlocks

InnerBlocks allow nesting blocks inside your custom block.

```jsx
import { useBlockProps, useInnerBlocksProps, InnerBlocks } from '@wordpress/block-editor';

const TEMPLATE = [
    [ 'core/heading', { level: 2, placeholder: 'Section Title' } ],
    [ 'core/paragraph', { placeholder: 'Section content...' } ],
];

const ALLOWED_BLOCKS = [ 'core/heading', 'core/paragraph', 'core/image', 'core/buttons' ];

export default function Edit() {
    const blockProps = useBlockProps();
    const innerBlocksProps = useInnerBlocksProps( blockProps, {
        template: TEMPLATE,
        templateLock: 'insert',       // 'all' | 'insert' | 'contentOnly' | false
        allowedBlocks: ALLOWED_BLOCKS,
        orientation: 'vertical',      // 'vertical' | 'horizontal'
        renderAppender: InnerBlocks.ButtonBlockAppender,
    } );

    return <div { ...innerBlocksProps } />;
}

export function Save() {
    const blockProps = useBlockProps.save();
    const innerBlocksProps = useInnerBlocksProps.save( blockProps );
    return <div { ...innerBlocksProps } />;
}
```

### Template Lock Modes

| Lock | Effect |
|------|--------|
| `false` | No restrictions. Users add/remove/move freely. |
| `'insert'` | Cannot add or remove blocks. Can reorder and edit content. |
| `'all'` | Cannot add, remove, or reorder. Content editing only. |
| `'contentOnly'` | Hides block structure. Only content (text, media) is editable. |

### Parent/Child Relationships

In the child block's `block.json`:

```json
{ "parent": [ "zentratec/section" ] }
```

The child only appears in the inserter when inside the parent.

---

## 6. Block Variations

Variations create pre-configured versions of an existing block.

### In block.json

```json
{
  "variations": [
    {
      "name": "testimonial",
      "title": "Testimonial",
      "description": "A quote styled as a testimonial.",
      "icon": "format-quote",
      "isDefault": false,
      "attributes": { "style": "testimonial", "showAvatar": true },
      "scope": [ "inserter", "block", "transform" ],
      "isActive": [ "style" ]
    }
  ]
}
```

### JS Registration

```js
import { registerBlockVariation } from '@wordpress/blocks';

registerBlockVariation( 'core/group', {
    name: 'zentratec-card',
    title: 'Card',
    description: 'A card container with padding and border.',
    icon: 'id-alt',
    attributes: {
        style: { border: { radius: '8px', width: '1px', color: '#e0e0e0' }, spacing: { padding: { top: '24px', right: '24px', bottom: '24px', left: '24px' } } },
        backgroundColor: 'white',
    },
    innerBlocks: [
        [ 'core/heading', { level: 3 } ],
        [ 'core/paragraph' ],
    ],
    scope: [ 'inserter' ],
} );
```

---

## 7. Block Patterns

Patterns are pre-built block layouts users insert from the inserter.

### PHP Registration

```php
function zentratec_register_patterns(): void {
    register_block_pattern_category( 'zentratec', [
        'label' => __( 'Zentratec', 'zentratec' ),
    ] );

    register_block_pattern( 'zentratec/cta-section', [
        'title'       => __( 'CTA Section', 'zentratec' ),
        'description' => __( 'A call-to-action with heading and button.', 'zentratec' ),
        'categories'  => [ 'zentratec', 'call-to-action' ],
        'keywords'    => [ 'cta', 'action', 'button' ],
        'blockTypes'  => [ 'core/group' ],
        'content'     => '<!-- wp:group {"align":"full","backgroundColor":"primary","layout":{"type":"constrained"}} -->
            <div class="wp-block-group alignfull has-primary-background-color has-background">
                <!-- wp:heading {"textAlign":"center","textColor":"white"} -->
                <h2 class="has-text-align-center has-white-color has-text-color">Ready to get started?</h2>
                <!-- /wp:heading -->
                <!-- wp:buttons {"layout":{"type":"flex","justifyContent":"center"}} -->
                <div class="wp-block-buttons">
                    <!-- wp:button {"backgroundColor":"white","textColor":"primary"} -->
                    <div class="wp-block-button"><a class="wp-block-button__link has-primary-color has-white-background-color has-text-color has-background">Contact Us</a></div>
                    <!-- /wp:button -->
                </div>
                <!-- /wp:buttons -->
            </div>
            <!-- /wp:group -->',
    ] );
}
add_action( 'init', 'zentratec_register_patterns' );
```

### File-based Patterns (WP 6.0+)

Place `.php` files in `patterns/` within your theme or plugin:

```php
<?php
/**
 * Title: Hero Section
 * Slug: zentratec/hero-section
 * Categories: zentratec, featured
 * Keywords: hero, banner
 * Block Types: core/group
 */
?>
<!-- wp:cover {"dimRatio":50,"minHeight":600} -->
<!-- /wp:cover -->
```

---

## 8. Block Supports

Supports provide design controls without custom code. Declared in `block.json`:

```json
{
  "supports": {
    "color": {
      "background": true,
      "text": true,
      "gradients": true,
      "link": true
    },
    "typography": {
      "fontSize": true,
      "lineHeight": true,
      "fontFamily": true,
      "fontWeight": true,
      "fontStyle": true,
      "textTransform": true,
      "letterSpacing": true,
      "textDecoration": true
    },
    "spacing": {
      "margin": true,
      "padding": true,
      "blockGap": true
    },
    "dimensions": {
      "minHeight": true
    },
    "border": {
      "color": true,
      "radius": true,
      "style": true,
      "width": true
    },
    "layout": {
      "default": { "type": "constrained" },
      "allowSwitching": true
    },
    "anchor": true,
    "align": [ "wide", "full" ],
    "html": false,
    "className": true,
    "customClassName": true
  }
}
```

Supports generate CSS custom properties and classes automatically. Use `get_block_wrapper_attributes()` in dynamic blocks to output them.

---

## 9. Build Pipeline

### Project Setup

```bash
# Scaffold a new block plugin
npx @wordpress/create-block@latest zentratec-blocks --namespace zentratec

# Or add wp-scripts to an existing project
bun add -D @wordpress/scripts
```

### package.json Scripts

```json
{
  "scripts": {
    "build": "wp-scripts build",
    "start": "wp-scripts start",
    "lint:js": "wp-scripts lint-js",
    "lint:css": "wp-scripts lint-style",
    "test:unit": "wp-scripts test-unit-js",
    "test:e2e": "wp-scripts test-e2e"
  }
}
```

### Multi-Block webpack Configuration

```js
// webpack.config.js
const defaultConfig = require( '@wordpress/scripts/config/webpack.config' );
const path = require( 'path' );

module.exports = {
    ...defaultConfig,
    entry: {
        'hero-banner/index': path.resolve( __dirname, 'src/blocks/hero-banner/index.js' ),
        'recent-posts/index': path.resolve( __dirname, 'src/blocks/recent-posts/index.js' ),
        'cta-card/index': path.resolve( __dirname, 'src/blocks/cta-card/index.js' ),
    },
    output: {
        ...defaultConfig.output,
        path: path.resolve( __dirname, 'build/blocks' ),
    },
};
```

`wp-scripts build` generates `*.asset.php` files alongside each entry point, listing dependencies and a version hash. These are consumed automatically by `register_block_type()`.

---

## 10. Block Transforms

Transforms let users convert blocks between types.

```js
import { createBlock } from '@wordpress/blocks';

const transforms = {
    from: [
        {
            type: 'block',
            blocks: [ 'core/paragraph' ],
            transform: ( { content } ) => createBlock( 'zentratec/callout', { text: content } ),
        },
        {
            type: 'prefix',
            prefix: '!!',
            transform: ( content ) => createBlock( 'zentratec/callout', { text: content } ),
        },
        {
            type: 'enter',
            regExp: /^!!\s?$/,
            transform: () => createBlock( 'zentratec/callout' ),
        },
        {
            type: 'raw',
            isMatch: ( node ) => node.nodeName === 'BLOCKQUOTE' && node.classList.contains( 'callout' ),
            transform: ( node ) => createBlock( 'zentratec/callout', { text: node.textContent } ),
        },
    ],
    to: [
        {
            type: 'block',
            blocks: [ 'core/paragraph' ],
            transform: ( { text } ) => createBlock( 'core/paragraph', { content: text } ),
        },
    ],
};

// Include in registerBlockType or block.json edit config
registerBlockType( 'zentratec/callout', { transforms, edit: Edit, save: Save } );
```

---

## 11. Testing Blocks

### Unit Tests (Jest)

```js
// src/blocks/hero-banner/__tests__/edit.test.js
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Edit from '../edit';

// Mock block-editor hooks
jest.mock( '@wordpress/block-editor', () => ( {
    useBlockProps: () => ( { className: 'test-block' } ),
    RichText: ( { value, onChange, placeholder } ) => (
        <input value={ value } onChange={ ( e ) => onChange( e.target.value ) } placeholder={ placeholder } />
    ),
    InspectorControls: ( { children } ) => <div data-testid="inspector">{ children }</div>,
    BlockControls: ( { children } ) => <div data-testid="toolbar">{ children }</div>,
} ) );

describe( 'HeroBanner Edit', () => {
    const defaultProps = {
        attributes: { heading: '', mediaUrl: '', overlayOpacity: 50 },
        setAttributes: jest.fn(),
    };

    it( 'renders heading input with placeholder', () => {
        render( <Edit { ...defaultProps } /> );
        expect( screen.getByPlaceholderText( /enter heading/i ) ).toBeInTheDocument();
    } );

    it( 'calls setAttributes when heading changes', async () => {
        render( <Edit { ...defaultProps } /> );
        await userEvent.type( screen.getByPlaceholderText( /enter heading/i ), 'Hello' );
        expect( defaultProps.setAttributes ).toHaveBeenCalled();
    } );
} );
```

### E2E Tests (Playwright)

```js
// e2e/hero-banner.spec.js
import { test, expect } from '@wordpress/e2e-test-utils-playwright';

test.describe( 'Hero Banner Block', () => {
    test( 'can be inserted and edited', async ( { admin, editor, page } ) => {
        await admin.createNewPost();
        await editor.insertBlock( { name: 'zentratec/hero-banner' } );

        const block = page.locator( '[data-type="zentratec/hero-banner"]' );
        await expect( block ).toBeVisible();

        await block.locator( 'h1[contenteditable]' ).fill( 'Test Heading' );
        await expect( block.locator( 'h1' ) ).toHaveText( 'Test Heading' );
    } );

    test( 'saves and renders on frontend', async ( { admin, editor, page } ) => {
        await admin.createNewPost();
        await editor.insertBlock( { name: 'zentratec/hero-banner', attributes: { heading: 'Live Test' } } );
        const postId = await editor.publishPost();

        await page.goto( `/?p=${ postId }` );
        await expect( page.locator( '.zentratec-hero h1' ) ).toHaveText( 'Live Test' );
    } );
} );
```

### Dynamic Block PHP Output Tests

```php
// tests/php/test-recent-posts-block.php
class Test_Recent_Posts_Block extends WP_UnitTestCase {
    public function test_render_with_posts(): void {
        $this->factory->post->create( [ 'post_title' => 'Test Post' ] );
        $output = zentratec_render_recent_posts( [ 'count' => 1 ], '', new WP_Block( [] ) );

        $this->assertStringContainsString( 'Test Post', $output );
        $this->assertStringContainsString( 'zentratec-recent-posts', $output );
    }

    public function test_render_empty_state(): void {
        $output = zentratec_render_recent_posts( [ 'count' => 3 ], '', new WP_Block( [] ) );
        $this->assertStringContainsString( 'No posts found', $output );
    }
}
```

---

## 12. Performance

### Conditional Asset Loading

Use `viewScript` in `block.json` instead of `script`. WordPress 6.5+ only enqueues `viewScript` when the block appears on the page.

```json
{
  "viewScript": "file:./view.js",
  "viewStyle": "file:./style-index.css"
}
```

### Lazy Loading Interactive Scripts (WP 6.5+ Interactivity API)

```json
{
  "viewScriptModule": "file:./view.js",
  "supports": { "interactivity": true }
}
```

### Dynamic Block Caching

```php
function zentratec_render_recent_posts( array $attributes, string $content, WP_Block $block ): string {
    $cache_key  = 'zentratec_recent_' . md5( wp_json_encode( $attributes ) );
    $cache_html = get_transient( $cache_key );

    if ( false !== $cache_html ) {
        return $cache_html;
    }

    // ... build $output ...

    set_transient( $cache_key, $output, HOUR_IN_SECONDS );
    return $output;
}
```

Invalidate on post publish:

```php
add_action( 'transition_post_status', function ( string $new, string $old ): void {
    if ( $new === 'publish' || $old === 'publish' ) {
        global $wpdb;
        $wpdb->query( "DELETE FROM {$wpdb->options} WHERE option_name LIKE '_transient_zentratec_recent_%'" );
    }
}, 10, 2 );
```

### Avoid Enqueuing Assets When Block Is Not Present

For blocks registered without `block.json` auto-enqueue, check before loading:

```php
function zentratec_conditionally_enqueue(): void {
    if ( has_block( 'zentratec/hero-banner' ) ) {
        wp_enqueue_style( 'zentratec-hero-style', plugins_url( 'build/hero-banner/style.css', __FILE__ ) );
    }
}
add_action( 'wp_enqueue_scripts', 'zentratec_conditionally_enqueue' );
```
