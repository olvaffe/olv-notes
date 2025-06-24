Python
======

## Installation

- install python
  - `pacman -S python`
- create per-user venv
  - `python -m venv --system-site-packages ~/.pip`
  - `export PATH="$HOME/.pip/bin:$PATH"`
  - to upgrade venv, `/usr/bin/python -m venv --upgrade ~/.pip`

## pip

- `pip list` lists installed packages
  - `pip list --outdated` lists outdated packages
- `pip install` installs specified packages
  - `pip install --upgrade pip`
  - meson requires `packaging` for version detection

## Tooling

- <https://github.com/astral-sh/uv>
- <https://github.com/astral-sh/ruff>

## Top GitHub Repos

- <https://github.com/ytdl-org/youtube-dl>, download youtube videos
- <https://github.com/yt-dlp/yt-dlp>, download youtube videos
- <https://github.com/django/django>, (server-side) web framework
- <https://github.com/fastapi/fastapi>, (server-side) web framework
- <https://github.com/pallets/flask>, (server-side) web framework
- <https://github.com/ansible/ansible>, IT automation
- <https://github.com/psf/requests>, http requests
- <https://github.com/psf/black>, python formatter
- <https://github.com/mitmproxy/mitmproxy>, http proxy for debugging
