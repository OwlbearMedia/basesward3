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

Pushing to `main` triggers the [Deploy via rsync](.github/workflows/deploy-scp.yml) GitHub Actions workflow, which deploys the site to the web server over SSH. Other branches are not deployed. The site is on shared cPanel hosting (LiteSpeed on CloudLinux), and the process matches the one used for voteforjulia.com on the same host.

Deploys never write into the live document root. Visitors see either the old site or the new one, never a half-uploaded mix. With the web root at `public_html`, each deploy:

1. Verifies the server's SSH host key against `SSH_HOST_FINGERPRINT` and refuses to connect if it doesn't match.
2. Uploads the site into a clean `public_html_next` directory next to the web root.
3. Copies any cPanel IP Blocker rules from the live `.htaccess` into the staged one (see below).
4. Swaps `public_html_next` into place, keeping the previous build as `public_html_prev`.
5. Checks that https://basesward3.com/ responds.

The swap uses Linux's `renameat2(RENAME_EXCHANGE)` (through `python3`) to exchange the two directories in one atomic step. If the host doesn't support it, the workflow falls back to two renames (`public_html` → `public_html_prev`, then `public_html_next` → `public_html`), which leaves the web root missing for a fraction of a second. The Actions log says which one ran.

`public_html` stays a real directory. It isn't replaced with a symlink to a release directory, because cPanel manages the document root and can reset it.

Each build contains exactly what's in the repo, so a file deleted from the repo is gone from the site after the next deploy. `.git*`, `.github`, `README.md`, and `LICENSE` are never uploaded.

### cPanel IP Blocker rules

cPanel's IP Blocker saves blocks as `deny from` lines in `public_html/.htaccess`, and builds its list from that file. The repo doesn't have those lines, so the deploy reads them from the live `.htaccess` and appends them to the staged one before the swap. Otherwise every deploy would silently lift every block. Manage blocks in cPanel → IP Blocker as usual; nothing about them belongs in this repo. The Actions log shows how many rules were carried over, never the addresses.

### Repository secrets

The workflow needs these repository secrets (**Settings → Secrets and variables → Actions**):

| Secret | Description |
| --- | --- |
| `SSH_HOST` | Server hostname only, e.g. `example.com` (no protocol, user, or path) |
| `SSH_PORT` | SSH port |
| `SSH_USERNAME` | SSH user |
| `SSH_PRIVATE_KEY` | Private key authorized on the server |
| `SSH_PASSPHRASE` | Passphrase for the private key |
| `SSH_TARGET_PATH` | The web root: `public_html` (relative to the SSH user's home) or an absolute path |
| `SSH_HOST_FINGERPRINT` | SHA256 fingerprint of the server's SSH host key, e.g. `SHA256:StI193FHo9…` |

The deploy won't run without `SSH_HOST_FINGERPRINT`. To get it, read it from your own `known_hosts` for a host you've already connected to and trust. CageFS hides `/etc/ssh` on the server, so it can't be read there:

```sh
ssh-keygen -F "[<host>]:<port>" -l | grep -v '^#'
```

Store the `SHA256:…` token itself. Any of the host's key types (ECDSA, Ed25519, RSA) works. If the host key ever changes, deploys fail with "The host presented no key matching SSH_HOST_FINGERPRINT" until the secret is updated.

### Rolling back

Swap the previous build back in over SSH:

```sh
cd ~ && mv public_html public_html_next && mv public_html_prev public_html
```

The next push to `main` deploys a new build on top as usual, and replaces `public_html_next`.

## License

[MIT](LICENSE)
