# lerni/silverstripe-assets-intervention4

Runs `silverstripe/assets` 3.2.x on **Intervention Image 4** by patching the package via
[cweagans/composer-patches](https://github.com/cweagans/composer-patches) (v2).

## Motivation

ImageMagick (`ext-imagick`) works well, but requires too much memory and pixel-cache resources when generating many image variants or reading dimensions on the fly during a request (often hitting pixel-cache limits or OOM crashes).

**`libvips`** (`intervention/image-driver-vips`) is significantly faster and uses a fraction of the memory, but making it work reliably with Silverstripe's asset pipeline requires **Intervention Image 4** (`intervention/image-driver-vips` ^4.2) along with a few small adjustments to `silverstripe/assets` (reading from streams instead of temp files, releasing resources after cold-cache dimension lookups, and handling `libvips` sequential pipelines across chained manipulations/conversions).

## What it does

- `provide`s `intervention/image` `3.11.99` to satisfy `silverstripe/assets`'s `intervention/image ^3.9` requirement while
  the real `intervention/image ^4.4` (and `intervention/gif ^5`) is installed. No inline alias in the root `composer.json`
  is needed (Composer does not apply inline aliases to tagged releases).
- Declares patches for `silverstripe/assets` in `extra.patches` of `composer.json` (fetched by `cweagans/composer-patches`
  from raw GitHub URLs pinned to the release tag and verified via `sha256`):

| Patch | Upstream status / target | Content |
|---|---|---|
| `0001-stream-read-no-tempfile.patch` | `silverstripe/silverstripe-assets` ([#716](https://github.com/silverstripe/silverstripe-assets/pull/716), in 3.4 / SS 6.4) | Reads the image from the asset stream instead of a temp file (commits `e0d93ec` + `5f11cc6`). Drop once `silverstripe/assets >= 3.4`. |
| `0002-intervention-image-4.patch` | `silverstripe/silverstripe-assets` | Intervention Image 4 port of `InterventionBackend`, `InterventionManagerFactory`, `InterventionImageFileConverter` and `composer.json`. |
| `0003-unload-resource-after-dimension-cache-miss.patch` | `silverstripe/silverstripe-assets` | Unloads the decoded `InterventionImage` resource (`setImageResource(null)`) after warming the dimension cache on a cache miss in `InterventionBackend::getDimensions()`, preventing unbounded memory retention during bulk dimension lookups. Note: if a manipulation runs on the same backend instance immediately after a cold-cache dimension lookup, the image will be decoded a second time (whereas `manipulateImage()` loads the resource before `getDimensions()` is called, so normal `FillMax`/`Pad` calls are unaffected). |
| `0004-vips-ensure-in-memory.patch` | `Intervention/image-driver-vips` / `silverstripe/silverstripe-assets` | Calls `Intervention\Image\Drivers\Vips\Core::ensureInMemory()` before encoding (`writeToStore()` / `writeTo()`) and before cloning a modified resource (`createCloneWithResource()`). When using `intervention/image-driver-vips`, sources are opened in `Vips\Access::SEQUENTIAL` mode for single-pass streaming; if a manipulated `DBFile` hands its `Image_Backend` on to a chained manipulation or format conversion (e.g. `$Image.ScaleMaxHeight(284).Convert('webp')`), the first encode consumes the sequential pipeline and the second fails with `VIPS-WARNING: error in tile 0 x 0`. Rendering the already-scaled pipeline to memory before encoding/cloning allows arbitrary chained encodes and manipulations. |
| `upstream/tests-intervention-image-4.patch` | `silverstripe/silverstripe-assets` (not applied by Composer) | Test changes (new tests for `writeTo()` quality, `paddedResize()` transparency, failure classification, unloading the resource after a `getDimensions()` cache miss, and chained manipulations/conversions). **Not applied by Composer** because the dist archive of `silverstripe/assets` ships no `tests/`. Kept in the repo for upstream PRs. |

## Install

Add the VCS repository to your project's `composer.json` (not published on Packagist), allow `cweagans/composer-patches`, and require the package:

```bash
composer config repositories.silverstripe-assets-intervention4 vcs https://github.com/lerni/silverstripe-assets-intervention4
composer config allow-plugins.cweagans/composer-patches true
composer require lerni/silverstripe-assets-intervention4:^3.2
```

Commit `patches.lock.json` next to `composer.lock`. Subsequent `composer install` runs (e.g. during deployment) automatically download and apply the locked patches to `silverstripe/assets`.

### Updating patches in a consuming project

Because `cweagans/composer-patches` v2 locks patch URLs and hashes in the project's `patches.lock.json`, updating to a new release of `lerni/silverstripe-assets-intervention4` requires relocking and repatching:

```bash
composer update lerni/silverstripe-assets-intervention4
composer patches-relock
composer patches-repatch
```

### Using the `libvips` driver

Install `intervention/image-driver-vips` (`^4.2`) and configure `InterventionImageDriver` in a YAML block ordered **after** `#assetsimage-imagick` (otherwise Silverstripe's conditional `imagick` block takes precedence when `ext-imagick` is loaded):

```bash
composer require intervention/image-driver-vips:^4.2
```

```yaml
---
Name: app-image-backend
After:
  - '#assetsimage'
  - '#assetsimage-imagick'
---
SilverStripe\Core\Injector\Injector:
  InterventionImageDriver:
    class: Intervention\Image\Drivers\Vips\Driver
```

## Project code that has to change for Intervention Image 4

If custom project code interacts directly with `Intervention\Image` classes or resources, update v3 calls to v4:

- `ImageManager::withDriver($driver, ...)` -> `new ImageManager($driver, ...)`
- `$manager->driver()` -> `$manager->driver` (public property)
- `$manager->read(...)` -> `$manager->decode(...)` / `decodePath(...)` / `decodeStream(...)`
- `$resource->encodeByExtension(...)` -> `$resource->encodeUsingFileExtension(...)`
- `$resource->place(...)` -> `$resource->insert(...)`
- `$resource->pickColor(...)` -> `$resource->colorAt(...)`
- `Color::parse('transparent')` throws in v4 (unlike v3's `Color::create('transparent')`). Hex (`ffffff`, `#fff`, `#00000000`), named colours and `rgba(0, 0, 0, 0)` all parse as expected; in Silverstripe templates/code using `Pad()`, pass a hex colour + `$transparencyPercent = 100` (e.g. `$Image.Pad(200, 200, 'FFFFFF', 100)`) or `'rgba(0,0,0,0)'` instead of `'transparent'`.

See the [Intervention Image v4 upgrade guide](https://image.intervention.io/v4/getting-started/upgrade).

## Releasing a new version of this package

Dependency patches in `cweagans/composer-patches` must use public URLs, because local file paths in `extra.patches` are resolved against the consuming project's root directory rather than this package's directory inside `vendor/`. Therefore, the `url` entries in `composer.json` point to raw GitHub URLs pinned to a release tag (`https://raw.githubusercontent.com/lerni/silverstripe-assets-intervention4/<tag>/patches/...`) together with each file's `sha256` checksum.

When adding or modifying a patch in `patches/`:

1. Update the `.patch` files in `patches/`.
2. Compute checksums: `sha256sum patches/*.patch`.
3. Update the `sha256` values and bump the tag in the raw GitHub URLs in `composer.json` (e.g. `3.2.1` -> `3.2.2`).
4. Commit and push to `main`, then create and push the matching git tag:
   ```bash
   git tag 3.2.2
   git push origin main --tags
   ```
   *(The tag must exist on GitHub before consumers run `composer update` / `composer patches-relock`, as Composer fetches the patch files from the tagged raw URL.)*

## Regenerating the patches

Install `silverstripe/assets` as a source checkout (`composer reinstall silverstripe/assets --prefer-source`), commit the
change set on a branch, and export with `git diff <base> <head> -- src composer.json`.
Back up the branch first: `composer patches-repatch` replaces the package directory.

## Removing

Remove this package and run `composer update intervention/image intervention/gif silverstripe/assets` once upstream
`silverstripe/assets` natively supports Intervention Image 4.
