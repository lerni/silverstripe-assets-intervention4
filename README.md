# lerni/silverstripe-assets-intervention4

Runs `silverstripe/assets` 3.2.x on **Intervention Image 4** by patching the package with
[cweagans/composer-patches](https://github.com/cweagans/composer-patches) (v2).

## What it does

- `provide`s `intervention/image` 3.11.99, so `silverstripe/assets` (`^3.9`) is satisfied while the real
  `intervention/image ^4.4` (and `intervention/gif ^5`) is installed. No inline alias in the root `composer.json` is needed
  (Composer does not apply inline aliases to tagged releases).
- Applies the patches in `patches/` to `silverstripe/assets`:

| Patch | Content |
|---|---|
| `0001-stream-read-no-tempfile.patch` | Reads the image from the asset stream instead of a temp file (upstream PR [#716](https://github.com/silverstripe/silverstripe-assets/pull/716), commits `e0d93ec` + `5f11cc6`, expected in assets 3.4 / Silverstripe 6.4). Drop it once `silverstripe/assets >= 3.4`. |
| `0002-intervention-image-4.patch` | Intervention 4 port of `InterventionBackend`, `InterventionManagerFactory`, `InterventionImageFileConverter` and `composer.json`. |
| `0003-unload-resource-after-dimension-cache-miss.patch` | Unloads the decoded `InterventionImage` resource (`setImageResource(null)`) after warming the dimension cache on a cache miss in `InterventionBackend::getDimensions()`, preventing unbounded memory retention during bulk dimension lookups. |
| `0004-vips-ensure-in-memory.patch` | Calls `Intervention\Image\Drivers\Vips\Core::ensureInMemory()` before encoding (`writeToStore()` / `writeTo()`) and before cloning a modified resource (`createCloneWithResource()`). When using `intervention/image-driver-vips`, sources are opened in `Vips\Access::SEQUENTIAL` mode for single-pass streaming; if a manipulated `DBFile` hands its `Image_Backend` on to a chained manipulation or format conversion (e.g. `$Image.ScaleMaxHeight(284).Convert('webp')`), the first encode consumes the sequential pipeline and the second fails with `VIPS-WARNING: error in tile 0 x 0`. Rendering the already-scaled pipeline to memory before encoding/cloning allows arbitrary chained encodes and manipulations. |
| `upstream/0003-intervention-image-4-tests.patch` | Test changes (new tests for `writeTo()` quality, `paddedResize()` transparency, failure classification, unloading the resource after a `getDimensions()` cache miss, and chained manipulations/conversions). **Not applied by Composer**: the dist archive of `silverstripe/assets` ships no `tests/`. Kept for an upstream PR. |

## Install

```bash
composer config allow-plugins.cweagans/composer-patches true
composer require lerni/silverstripe-assets-intervention4
```

Commit `patches.lock.json` next to `composer.lock`. Afterwards `composer install` re-applies the patches.

## Project code that has to change with it

- `ImageManager::withDriver()` is gone -> `new ImageManager($driver, ...)`.
- `$manager->driver()` is now the public property `$manager->driver`.
- `Image::pickColor()` is now `colorAt()`, `read()` is `decode*()`, `encodeByExtension()` is `encodeUsingFileExtension()`.

See the [Intervention v4 upgrade guide](https://image.intervention.io/v4/getting-started/upgrade).

## Releasing

Dependency patches must use public URLs, because composer-patches resolves local paths against the project root, not
the dependency. The `url` values in `composer.json` therefore point at raw GitHub URLs pinned to a tag
(`.../3.2.0/patches/...`) together with the `sha256` of each file.

When a patch changes: update the files in `patches/`, run `sha256sum patches/*.patch`, put the hashes and the new tag
into `composer.json`, commit, then create that tag (e.g. `3.2.1`). The tag must be pushed before consumers can install.

## Regenerating the patches

Install `silverstripe/assets` as a source checkout (`composer reinstall silverstripe/assets --prefer-source`), commit the
change set on a branch, and export with `git diff <base> <head> -- src composer.json`.
Back up the branch first: `composer patches-repatch` replaces the package directory.

## Removing

Remove the module and `composer update intervention/image intervention/gif silverstripe/assets` after the upstream
`silverstripe/assets` supports Intervention 4.
