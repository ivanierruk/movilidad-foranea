# Contributing to movilidad-foranea

Thank you for your interest in contributing. This project extracts data from
live commercial websites, so contributions that keep the scrapers working as
those sites change are especially welcome.

## Reporting bugs and requesting features

Please open an issue on GitHub. For a bug, include:

- the platform affected (ado, omnibus, clickbus, busbud, booking),
- the command and the `config.yaml` you used (you can shorten it),
- what happened, including the relevant lines from
  `output/<platform>/scraping_errors_<PLATFORM>.log`,
- your operating system, Python version and Google Chrome version.

Because these scrapers depend on the current HTML of each site, a scraper that
stops returning data is often caused by a change on the platform's side (markup,
selectors, bot protection). Reports that pinpoint which selector or step broke
are very helpful.

## Setting up a development environment

Requires Python 3.10+ and Google Chrome.

```bash
git clone https://github.com/ivanierruk/movilidad-foranea.git
cd movilidad-foranea

# Reproducible environment (pinned). Using uv:
uv venv
uv pip install -r requirements.txt
uv pip install -e . --no-deps
uv pip install pytest
```

The full dependency stack is pinned in `requirements.txt` on purpose:
`selenium-wire` (used by the Booking scraper) is unmaintained and breaks with a
newer TLS/HTTP stack. Install from `requirements.txt` to get a known-good
environment.

## Running the tests

```bash
# Unit tests: pure logic, no browser or network. These run in CI.
pytest

# Integration tests: launch Chrome and check the platform URLs. Run on demand;
# they need a browser and network.
pytest -m integration
```

Please make sure `pytest` (the unit tests) passes before opening a pull request.
Continuous integration runs the unit tests on every push and pull request, and
must be green before a change is merged.

## Proposing a change

1. Fork the repository and create a branch for your change.
2. Keep each commit a working, self-contained unit.
3. Add or update tests when you change behaviour.
4. Make sure `pytest` passes and (if relevant) `pytest -m integration` too.
5. Open a pull request describing what you changed and why.

## Adding a new platform

See the "Adding a new platform" section of
[`docs/architecture.md`](docs/architecture.md). In short: add a module under
`mobility/` (or `accommodation/`) with a scraper class and a `scrape_<name>()`
function, register it in `cli.py`, and reuse the shared helpers
(`build_travel_dates`, `normalize_missing`, `validate_*`, `setup_file_logging`,
`build_retryer`).

## Code style

- Code, comments and docstrings are written in **English**.
- Follow the structure and naming of the existing modules.
- Keep configuration external (in `config.yaml`); importing a module must never
  start a download or an extraction.

## License

By contributing, you agree that your contributions will be licensed under the
project's license, the GNU General Public License v3.0 only (GPL-3.0-only).
