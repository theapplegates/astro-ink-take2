# Cloudinary `<Picture>` workflow

How to add a responsive, multi-format (JXL → AVIF → WebP) image to a blog post.
Images are served entirely by Cloudinary — Astro/Sharp never processes them.

## Prerequisites (one time)

- **Node 22+** must be active. This project requires it (Astro 6 refuses Node 20).
  ```bash
  nvm use 22
  ```
- **Use npm**, not pnpm/bun (peer-dep resolution only works with npm here).
- `.env` must contain your Cloudinary credentials (already set):
  - `CLOUDINARY_CLOUD_NAME` (or `PUBLIC_CLOUDINARY_CLOUD_NAME`)
  - `CLOUDINARY_API_KEY`
  - `CLOUDINARY_API_SECRET`

## Step 1 — Add the image

Drop the source file into:

```
src/assets/images/my-photo.jpg
```

## Step 2 — Generate breakpoints

```bash
npm run cloudinary:breakpoints -- src/assets/images/my-photo.jpg
```

> The `--` is required so npm passes the file path through to the script.
> You can pass several files at once.

This **uploads the image to Cloudinary** (overwriting any existing copy with the
same public ID), asks Cloudinary for responsive breakpoint widths, and merges the
result into `src/data/cloudinary-breakpoints.json`:

```json
{
  "assets/images/my-photo": [200, 382, 527, 730, 1024, 2000]
}
```

### The key rule

The **public ID / key has no file extension and no `src/` prefix**:

| File on disk                         | Key used everywhere        |
| ------------------------------------ | -------------------------- |
| `src/assets/images/my-photo.jpg`     | `assets/images/my-photo`   |

You must use this exact key in both `src` and the `breakpoints` lookup (Step 3).
A mismatch (e.g. leaving `.jpg` on) makes the lookup return `undefined` and the
build fails with "Cloudinary breakpoints are missing".

## Step 3 — Use it in a blog post (`src/content/blog/*.mdx`)

```mdx
import Picture from "@/components/Picture.astro";
import breakpoints from "@/data/cloudinary-breakpoints.json";

<Picture
  src="assets/images/my-photo"
  alt="Describe the image."
  width={2000}
  height={1500}
  sizes="(min-width: 768px) 720px, 100vw"
  breakpoints={breakpoints["assets/images/my-photo"]}
  pictureClass="responsive-picture"
/>
```

### `<Picture>` props

| Prop              | Required | Notes                                                        |
| ----------------- | -------- | ------------------------------------------------------------ |
| `src`             | yes      | Cloudinary public ID — the no-extension key from Step 2.     |
| `alt`             | yes      | Throws at build time if missing.                             |
| `width` / `height`| yes      | Intrinsic size; `width` is also added to the srcset widths.  |
| `sizes`           | yes      | Standard responsive `sizes` attribute.                       |
| `breakpoints`     | yes      | The array from `cloudinary-breakpoints.json` for this key.   |
| `pictureClass`    | no       | Class applied to the `<picture>` element.                    |
| `transformations` | no       | Extra Cloudinary options (merged into every generated URL).  |

The component outputs a `<picture>` with `<source>` elements for **JXL, AVIF, and
WebP** (in that order) plus a **WebP `<img>`** fallback. There is no JPEG fallback.

## Verify

```bash
npm run build && npm run preview
```

Open a post and check which format the browser picked:

```js
document.querySelector("picture img")?.currentSrc;
```

- Safari 17+ → `f_jxl`
- Chrome / Firefox → `f_avif`
- Older browsers → `f_webp`
- You should never see `f_jpg`.
