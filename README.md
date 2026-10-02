# tesseract

Container images with [Tesseract](https://tesseract-ocr.github.io/), the OCR engine, for reading printed text from scanned pages and photos on the command line. Tesseract is compiled from the signed release tag on Ubuntu and Alpine against the distro's Leptonica, with the English and the orientation and script detection models from tessdata_fast. `latest` also includes Poppler, for OCR of PDF files. The images are rebuilt when Tesseract publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the Tesseract project. Report problems with the image in this repository and problems with Tesseract itself [upstream](https://github.com/tesseract-ocr/tesseract/issues).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/tesseract page.png page
```

This reads `page.png` and writes the text to `page.txt`.

Make a searchable PDF, which keeps the page image and adds the recognized text as an invisible layer:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/tesseract page.png page pdf
```

The entrypoint runs `tesseract` under `tini` in `/work`, so file names are relative to the directory you mount. The arguments are the input, the output name without an extension, then options and output formats. A few more commands:

```sh
# Print the text of an image read from standard input
docker run --rm -i ghcr.io/randomcontainers/tesseract - - < page.png

# Write page.txt, page.hocr, page.tsv and page.pdf in one run
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/tesseract \
  page.png page txt hocr tsv pdf

# ALTO XML (out.xml) and PAGE XML (out.page.xml)
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/tesseract \
  page.png out alto page

# Read an image that holds a single line of text
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/tesseract \
  line.png - --psm 7

# Detect the orientation and script of a page, written to page.osd
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/tesseract \
  page.png page --psm 0

# Read every image listed in pages.txt, one file name per line, into one PDF
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/tesseract \
  pages.txt book pdf
```

Tesseract reads images, not PDF files. The default image has Poppler, whose `pdftoppm` renders the pages of a PDF first. It numbers the pages with leading zeros, so `ls` lists them in order:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint pdftoppm \
  ghcr.io/randomcontainers/tesseract -r 300 -gray -png input.pdf page
ls page-*.png > pages.txt
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/tesseract pages.txt searchable pdf
```

The [command-line usage page](https://tesseract-ocr.github.io/tessdoc/Command-Line-Usage.html) of the Tesseract documentation covers the options, page segmentation modes and output formats.

## What is in the image

| | slim | default |
|---|---|---|
| `tesseract` and the `libtesseract` shared library | yes | yes |
| The `eng` (English) and `osd` (orientation and script detection) models | yes | yes |
| Poppler: `pdftoppm`, `pdftotext`, `pdfimages`, `pdfinfo` and the other Poppler tools | no | yes |

Poppler is the [randomcontainers/poppler](https://github.com/randomcontainers/poppler) build.

Tesseract reads images through the distro's Leptonica: PNG, JPEG, TIFF including multi-page files, GIF, WebP, BMP and PNM, plus JPEG 2000 on Ubuntu. It is built with libcurl, so the input can also be an `http://` or `https://` URL. libarchive lets a `.traineddata` file be an archive of the model's parts, and OpenMP spreads the work on a page over several threads (see [Threads](#threads)). Both engines are built: the LSTM engine, which the `eng` model uses, and the legacy engine, which orientation and script detection (`--psm 0` and `--psm 1`) needs.

Not included: the training tools such as `lstmtraining` and `text2image`, which need Pango, Cairo and ICU; the C++ headers and pkg-config file for building against `libtesseract`; the manual pages; and models for other languages (see [Languages](#languages)). The configure options and the Leptonica version are in `/usr/local/share/randomcontainers/tesseract/buildinfo`.

## Default or slim

Use the default image (`latest`) when you OCR PDF files: Poppler renders their pages to images, and `pdftotext` and `pdfinfo` check the result. `slim` has Tesseract and the libraries it needs, without Poppler; use it to OCR images, or as the base of your own image, for example one with more languages.

The default image includes Poppler, which is licensed under the GNU General Public License (GPL-2.0-only OR GPL-3.0-only). Use `slim` if your policy excludes GPL software. The default image is also published as `ghcr.io/randomcontainers/tesseract-poppler`, built in the [tesseract-poppler](https://github.com/randomcontainers/tesseract-poppler) repository with the same contents and a different digest.

## Tags

`<version>` is a Tesseract release such as `5.5.3`. `<minor>` and `<major>` are its shorter forms, `5.5` and `5`, and follow the newest release in that series.

| Default (with Poppler) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Tesseract version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation. On amd64, Tesseract picks its SIMD code (SSE4.1, AVX, AVX2, FMA or AVX-512) at run time from what the CPU supports, so the same image runs on older and newer CPUs. On arm64 it uses NEON.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Languages

The images have two models from [tessdata_fast](https://github.com/tesseract-ocr/tessdata_fast), version 4.1.0: `eng` for English and `osd` for orientation and script detection. The same repository has models for more than 100 other languages and scripts, and [tessdata_best](https://github.com/tesseract-ocr/tessdata_best) has slower models that are often more accurate. The [data files page](https://tesseract-ocr.github.io/tessdoc/Data-Files.html) lists them.

To use another language without building an image, mount its model file into `/usr/local/share/tessdata/` and pick it with `-l`. Join several languages with `+`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  -v "$PWD/deu.traineddata:/usr/local/share/tessdata/deu.traineddata:ro" \
  ghcr.io/randomcontainers/tesseract page.png page -l deu+eng
```

`tesseract --list-langs` prints the models it finds. To add models to an image of your own, see [Extending the slim image](#extending-the-slim-image).

## Threads

Tesseract uses OpenMP to spread the work on one page over several threads. That makes a single page faster, but when you OCR many pages at once, with one process per CPU, the threads compete for the same CPUs and the whole job gets slower. Set `OMP_THREAD_LIMIT=1` for such jobs:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e OMP_THREAD_LIMIT=1 \
  ghcr.io/randomcontainers/tesseract page.png page
```

## Untrusted files

Tesseract decodes images with Leptonica and the distro's image libraries. For images from unknown sources, take away what the container does not need. `--network none` also stops Tesseract from fetching a URL given as the input:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --network none --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 1g --pids-limit 64 \
  ghcr.io/randomcontainers/tesseract untrusted.png out
```

Load `.traineddata` files only from sources you trust. The code that reads them has had memory-safety bugs.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no Poppler, so Poppler updates do not rebuild it. This adds the German model:

```dockerfile
FROM ghcr.io/randomcontainers/tesseract:slim-ubuntu@sha256:...
ADD --chmod=644 --checksum=sha256:<sha256 of the file> \
  https://raw.githubusercontent.com/tesseract-ocr/tessdata_fast/4.1.0/deu.traineddata \
  /usr/local/share/tessdata/
```

`ADD` writes the file as root whatever the `USER` is, and `--chmod=644` lets the image's user read it. To install distro packages, switch to `USER root` for the `RUN` step and back to `USER 1000:1000` after it; the packages Tesseract needs are listed in `/usr/local/share/randomcontainers/tesseract/runtime-deps`. On Alpine, start from `slim-alpine`. The entrypoint is `["tini", "--", "tesseract"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new Tesseract releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. The models are in `/usr/local/share/tessdata/`, which is where Tesseract looks by default. `/usr/local/share/randomcontainers/tesseract/` holds the version, the source, the build options, the license file and `runtime-deps`.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/tesseract:latest \
  --repo randomcontainers/tesseract --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/tesseract-poppler` are built in that repository, so verify them with `--repo randomcontainers/tesseract-poppler` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/tesseract:latest --format '{{ json .SBOM }}'
```

Tesseract releases have no source tarball, only a tag signed by Stefan Weil. Before compiling, the build clones the release tag, checks that it points to the commit recorded in `package.yml`, and checks its signature against his key in `keys/tesseract-release.gpg` (fingerprint `4923 6FEA 75C9 5D69 8EC2 B78A E08C 21D5 6774 50AD`). The two model files are checked against the SHA-256 values in `package.yml`.

## Updates

The project checks the releases of [tesseract-ocr/tesseract](https://github.com/tesseract-ocr/tesseract/releases) every 15 minutes and skips release candidates. A release is picked up once it is 24 hours old. The new version and the commit its tag points to are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new Poppler image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

The `eng` and `osd` models are pinned to a version in `package.yml` and do not follow tessdata_fast automatically. Updating them is a commit to `package.yml`, which rebuilds the images of the current Tesseract version.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  --build-arg TESSDATA_ENG_VERSION=<version> \
  --build-arg TESSDATA_ENG_SHA256=<sha256> \
  --build-arg TESSDATA_OSD_VERSION=<version> \
  --build-arg TESSDATA_OSD_SHA256=<sha256> \
  -t tesseract:local .
```

The `TESSDATA_*` values are the `version` and `sha256` of the `extra-artifacts` entries in `package.yml`. Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

Tesseract is licensed under the Apache License, version 2.0 (Apache-2.0), and so are the tessdata_fast models. Both repositories ship the same `LICENSE` file, which is in `/usr/local/share/randomcontainers/tesseract/licenses/`. `/usr/local/share/randomcontainers/tesseract/source` names the Tesseract commit that was compiled and the URLs and SHA-256 of the model files. The build applies no patches, and the Dockerfiles hold every configure option.

The default image adds Poppler, licensed under GPL-2.0-only OR GPL-3.0-only, with an MIT-licensed UTF-8 decoder. Its license files and corresponding source are described in the [poppler repository](https://github.com/randomcontainers/poppler#licenses). The libraries from Ubuntu or Alpine, such as Leptonica, libarchive and libcurl, keep their own licenses; the SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
