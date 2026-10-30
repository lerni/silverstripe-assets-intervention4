# lerni/silverstripe-assets-intervention4

Runs `silverstripe/assets` and `silverstripe/asset-admin` 3.3.x (assets >= 3.3.2) on **Intervention Image 4** by patching the packages via
[cweagans/composer-patches](https://github.com/cweagans/composer-patches) (v2). The 3.2.x line (tags `3.2.*`) additionally carries a dimension-cache memory patch that 3.3.2 ships upstream ([#718](https://github.com/silverstripe/silverstripe-assets/pull/718)).

## Motivation

ImageMagick (`ext-imagick`) works well, but requires too much memory and pixel-cache resources when generating many image variants or reading dimensions on the fly during a request (often hitting pixel-cache limits or OOM crashes).

**`libvips`** (`intervention/image-driver-vips`) is significantly faster and uses a fraction of the memory, but making it work reliably with Silverstripe's asset pipeline requires **Intervention Image 4** (`intervention/image-driver-vips` ^4.2) along with a few small adjustments to `silverstripe/assets` (reading from streams instead of temp files and handling `libvips` sequential pipelines across chained manipulations/conversions).

## What it does

- `provide`s `intervention/image` `3.11.99` to satisfy `silverstripe/assets`'s `intervention/image ^3.9` requirement while
  the real `intervention/image ^4.4` (and `intervention/gif ^5`) is installed. No inline alias in the root `composer.json`
  is needed (Composer does not apply inline aliases to tagged releases).
- Declares patches for `silverstripe/assets` and `silverstripe/asset-admin` in `extra.patches` of `composer.json` (fetched by `cweagans/composer-patches`
  from raw GitHub URLs pinned to the release tag and verified via `sha256`):

| Patch | Upstream status / target | Content |
|---|---|---|
| `0001-stream-read-no-tempfile.patch` | `silverstripe/silverstripe-assets` ([#716](https://github.com/silverstripe/silverstripe-assets/pull/716)), **merged into `3.x`**, released with 3.4 / SS 6.4 | Reads the image from the asset stream instead of a temp file (commits `e0d93ec` + `5f11cc6`). Drop once `silverstripe/assets >= 3.4`. |
| `0002-intervention-image-4.patch` | `silverstripe/silverstripe-assets` | Intervention Image 4 port of `InterventionBackend`, `InterventionManagerFactory`, `InterventionImageFileConverter` and `composer.json`. |
| `0003-unload-resource-after-dimension-cache-miss.patch` (**merged, no longer applied in 3.3.x**) | `silverstripe/silverstripe-assets` ([#718](https://github.com/silverstripe/silverstripe-assets/pull/718)), **merged into `3.3`**, released with 3.3.2. Still applied by the `3.2.*` tags. | Unloads the decoded `InterventionImage` resource (`setImageResource(null)`) after warming the dimension cache on a cache miss in `InterventionBackend::getDimensions()`, preventing unbounded memory retention during bulk dimension lookups. Note: if a manipulation runs on the same backend instance immediately after a cold-cache dimension lookup, the image will be decoded a second time (whereas `manipulateImage()` loads the resource before `getDimensions()` is called, so normal `FillMax`/`Pad` calls are unaffected). |
| `0004-vips-ensure-in-memory.patch` | `Intervention/image-driver-vips` / `silverstripe/silverstripe-assets` | Calls `Intervention\Image\Drivers\Vips\Core::ensureInMemory()` before encoding (`writeToStore()` / `writeTo()`) and before cloning a modified resource (`createCloneWithResource()`). When using `intervention/image-driver-vips`, sources are opened in `Vips\Access::SEQUENTIAL` mode for single-pass streaming; if a manipulated `DBFile` hands its `Image_Backend` on to a chained manipulation or format conversion (e.g. `$Image.ScaleMaxHeight(284).Convert('webp')`), the first encode consumes the sequential pipeline and the second fails with `VIPS-WARNING: error in tile 0 x 0`. Rendering the already-scaled pipeline to memory before encoding/cloning allows arbitrary chained encodes and manipulations. |
| `0005-asset-admin-image-editor-intervention-image-4.patch` | `silverstripe/silverstripe-asset-admin` (>= 3.3) | Intervention Image 4 port of `ImageEditor` (`Direction::HORIZONTAL`/`VERTICAL` for `flip()`, clockwise `rotate()`, `encodeUsingFileExtension()`, and `VipsCore::ensureInMemory()`). |
| `upstream/tests-intervention-image-4.patch` | `silverstripe/silverstripe-assets` (not applied by Composer) | Test changes for the port: `GDImageTest` / `ImagickImageTest` use the v4 `driver` property, plus new tests in `ImageTestBase` (`writeTo()` quality, `paddedResize()` transparency, failure classification, chained manipulations/conversions). **Not applied by Composer** because the dist archive of `silverstripe/assets` ships no `tests/`. Kept in the repo for upstream PRs. |

## Companion: `axllent/silverstripe-scaled-uploads`

Upstream `axllent/silverstripe-scaled-uploads` copies every upload to a file in `TEMP_PATH`, loads the image backend from that path and reads EXIF from it. That temp-file round trip does not fit the stream-based backend of patch `0001` (and `silverstripe/assets` >= 3.4). [This fork branch](https://github.com/lerni/silverstripe-scaled-uploads/tree/fix/read-source-from-asset-store) reads the source from the asset store instead (EXIF from a stream) and resets the backend's resource after rewriting the file. Until it is merged upstream, require it in the project:

```json
"repositories": [
    { "type": "vcs", "url": "https://github.com/lerni/silverstripe-scaled-uploads" }
],
"require": {
    "axllent/silverstripe-scaled-uploads": "dev-fix/read-source-from-asset-store as 2.3.99"
}
```

## Install

Add the VCS repository to your project's `composer.json` (not published on Packagist), allow `cweagans/composer-patches`, and require the package:

```bash
composer config repositories.silverstripe-assets-intervention4 vcs https://github.com/lerni/silverstripe-assets-intervention4
composer config allow-plugins.cweagans/composer-patches true
composer require lerni/silverstripe-assets-intervention4:^3.3
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

#### PHP requirements

libvips is called through PHP's FFI (`jcupitt/vips`), which only binds to the system library: `libvips.so.42` has to be installed on the server (Debian package `libvips42`, `libvips42t64` on Debian 13). Besides that the FFI extension has to be enabled.

```ini
extension=ffi.so
ffi.enable="true"
```

`ffi.enable` defaults to `preload`, which blocks FFI at runtime. On shared hosting put the lines where the host allows custom ini files and check that CLI and web use the same PHP version.

#### Project glue (recommended)

The module itself does not ship these; the ootstra uses them:

- [`ImageManagerFactory`](https://github.com/lerni/ootstra/blob/master/app/src/Utility/ImageManagerFactory.php) replaces `InterventionManagerFactory`. It builds `new ImageManager($driver, autoOrientation: true)` and avoids a `TypeError` in Intervention's `Config::prepareOptions()` when Silverstripe's Injector merges constructor arguments as integer-keyed entries and limits libvips threads, see below.

  ```yaml
  SilverStripe\Core\Injector\Injector:
    Intervention\Image\ImageManager:
      factory: '%$App\Utility\ImageManagerFactory'
  ```

- **`VIPS_CONCURRENCY`**: libvips starts one worker thread per CPU core, which exhausts the process limits of shared hosting (glib `Error creating thread`). The factory calls `Config::concurrencySet()` with the value of `VIPS_CONCURRENCY` from `.env` (default `1`). It has to be set this way because `.env` is not exported to the C environment, so libvips would never see it as a plain environment variable.

  ```dotenv
  VIPS_CONCURRENCY=2
  ```

- [`AvifFileConverter`](https://github.com/lerni/ootstra/blob/master/app/src/Conversion/AvifFileConverter.php) extends `InterventionImageFileConverter`: it applies a lower default quality for AVIF (`avif_quality`, AVIF at Q55 is about JPEG at Q75), logs the full chain of previous exceptions when a conversion fails (the core only logs the outer message, the libvips cause is in the previous ones), and falls back to the already scaled variant instead of an empty `src`.

  ```yaml
  SilverStripe\Core\Injector\Injector:
    SilverStripe\Assets\Conversion\InterventionImageFileConverter:
      class: App\Conversion\AvifFileConverter
  App\Conversion\AvifFileConverter:
    avif_quality: 50
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
