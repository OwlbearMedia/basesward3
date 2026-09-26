# [basesward3.com](https://basesward3.com/)
![image](https://raw.githubusercontent.com/OwlbearMedia/basesward3/refs/heads/main/img/jacob-bases-rect.png)

Campaign website for Jacob Bases, candidate for Mankato City Council Ward 3.

The site is a single static page: plain HTML, CSS, and JavaScript. There is no build step, no framework, and no package manager.

## Project structure

```
.
├── index.html                  # The entire site: markup, SEO meta tags, JSON-LD
├── css/styles.css              # All styles
├── js/scripts.js               # Mobile menu, header offset, form validation, click tracking
├── img/                        # Banner, logos, candidate and endorsement photos
├── robots.txt
├── sitemap.xml
└── .github/workflows/
    └── deploy-scp.yml          # Deploys to the web server on push to main
```

## Page sections

`index.html` is one page split into anchored sections, linked from the header nav:

| Section | Anchor | Notes |
| --- | --- | --- |
| Hero | `#home` | Banner image, preloaded for fast first paint |
| About Jacob | `#about` | Bio |
| Issues | `#issues` | Embedded YouTube videos (privacy-enhanced `youtube-nocookie.com`, lazy loaded) |
| Endorsements | `#endorsements` | Endorsement of Julia Hamann for Mayor, linking to voteforjulia.com |
| Letter of Support | `#letter-of-support` | Blue Earth County DFL letter of support |
| Donate | `#donate` | Donorbox donation widget |
| Volunteer | `#volunteer` | Contact form submitted to Formspree |

## Running locally

Open `index.html` in a browser, or serve the folder so relative paths and embeds behave like production:

```sh
python3 -m http.server 8000
# then visit http://localhost:8000
```

The Donorbox widget, YouTube embeds, Google Fonts, and Google Analytics load from external services, so they need an internet connection.

## Third-party services

| Service | Used for | Where it's configured |
| --- | --- | --- |
| Google Analytics (gtag.js) | Page analytics, outbound click events | `<head>` of `index.html` (ID `G-53PB7HPW81`); events in `js/scripts.js` |
| Donorbox | Donation form | `<dbox-widget campaign="...">` in the `#donate` section |
| Formspree | Contact / volunteer form | `action` attribute of `.contact-form` |
| YouTube | Issue videos | `<iframe>` embeds in the `#issues` section |
| Google Fonts | "Poetsen One" typeface | `<link>` in `<head>` |

### Analytics events

`js/scripts.js` sends an `outbound_click` event to Google Analytics when visitors click:

- the Julia Hamann endorsement image (`#julia-endorsement-link`, label `julia_homepage`)
- the Julia donate link (`#julia-donate-link`, label `julia_donate`)

To track another outbound link, give it an `id` and call `trackOutboundClick(label, url)` from its click handler.

## Making changes

- **Content:** edit `index.html` directly.
- **Styles:** edit `css/styles.css`, then bump the cache-busting version in `index.html` (`css/styles.css?ver=1.3.0`) so returning visitors get the new file.
- **Images:** add to `img/` and always set `width`, `height`, and `alt`. Use `loading="lazy"` for anything below the fold.
- **SEO:** the meta description, Open Graph and Twitter tags, and JSON-LD structured data in `<head>` all repeat the same campaign details. If you change the title, description, or images, update every copy. Update `<lastmod>` in `sitemap.xml` after significant content changes.
- **Footer:** the "Paid for by" disclaimer is required on campaign materials. Don't remove it.

## Deployment

Pushing to `main` triggers the [Deploy via rsync](.github/workflows/deploy-scp.yml) GitHub Actions workflow, which deploys the site to the web server over SSH. Other branches are not deployed.

Deploys are atomic: visitors see either the old site or the new one, never a half-uploaded mix. On the server, the web root (`SSH_TARGET_PATH`) is a symlink to the live release:

```
/var/www/basesward3.com            -> /var/www/basesward3.com-releases/20260926204859-ae10e35
/var/www/basesward3.com-releases/
    20260926204855-d294087/
    20260926204859-ae10e35/        # live
```

Each deploy:

1. Uploads the repo into a new directory in `<SSH_TARGET_PATH>-releases/`, named `<UTC timestamp>-<short commit SHA>`. Unchanged files are hard-linked from the live release, so only changed files are transferred.
2. Repoints the web root symlink at the new release with a single rename, which is atomic.
3. Deletes all but the 5 newest releases (`KEEP_RELEASES` in the workflow).

Each release contains exactly what's in the repo, so a file deleted from the repo is gone from the site after the next deploy. `.git*`, `.github`, `README.md`, and `LICENSE` are never uploaded.

### Repository secrets

The workflow needs these repository secrets (**Settings → Secrets and variables → Actions**):

| Secret | Description |
| --- | --- |
| `SSH_HOST` | Server hostname only, e.g. `example.com` (no protocol, user, or path) |
| `SSH_PORT` | SSH port |
| `SSH_USERNAME` | SSH user |
| `SSH_PRIVATE_KEY` | Private key authorized on the server |
| `SSH_PASSPHRASE` | Passphrase for the private key |
| `SSH_TARGET_PATH` | Absolute path to the web root on the server |

### Rolling back

Point the symlink at an older release:

```sh
cd /var/www   # the directory containing SSH_TARGET_PATH
ln -s basesward3.com-releases/<older-release> basesward3.com.tmp && mv -T basesward3.com.tmp basesward3.com
```

The next push to `main` deploys a new release on top as usual.

### Server requirements

- A Linux server (the swap uses GNU `mv -T`) with `bash` and `rsync`.
- The SSH user must be able to write to the parent directory of `SSH_TARGET_PATH`, because the releases directory and the symlink live there.
- The web server must follow a symlinked document root. nginx and Apache do by default.

### First deploy

The first deploy finds a real directory at `SSH_TARGET_PATH` instead of a symlink. A plain rename can't replace a directory with a symlink, so the workflow uses Linux's `renameat2(RENAME_EXCHANGE)` (through `python3`) to swap the two in one atomic step, with no downtime. The old directory is kept at `<SSH_TARGET_PATH>.pre-atomic-<release>`; delete it once the new setup is confirmed working.

If the server has no `python3`, or the filesystem doesn't support the exchange, the deploy fails with the live site unchanged. Migrate by hand once over SSH instead. The site is briefly unavailable between the two commands:

```sh
cd /var/www   # the directory containing SSH_TARGET_PATH
mkdir -p basesward3.com-releases
mv basesward3.com basesward3.com-releases/00000000000000-initial && ln -s basesward3.com-releases/00000000000000-initial basesward3.com
```

Then re-run the workflow.

## License

[MIT](LICENSE)
