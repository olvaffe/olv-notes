# Python

## Installation

- arch: `pacman -S python`
- debian: `apt install python3 python3-venv`

## venv

- python standard library includes `venv` and `ensurepip` modules
- `python -m venv <dir>` creates a new venv
  - `--without-pip` skips pip in the new venv
  - `--system-site-packages` allows the new venv to use system packages
- the convention is to use per-project venvs
  - `cd project`
  - `python -m venv .venv`
  - `source .venv/bin/activate`
  - `pip install -r requirements.txt`
  - `pip freeze > requirements.txt`
- per-user venv
  - `python -m venv --system-site-packages ~/.local/share/venv`
  - `export PATH="$HOME/.local/share/venv/bin:$PATH"`
  - to upgrade venv after system python upgrade,
    `/usr/bin/python -m venv --upgrade --system-site-packages ~/.local/share/venv`

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
