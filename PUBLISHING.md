# Publishing `capturethenarrative` to PyPI

Everything happens from the root of this repository.

## One-time setup

1. Create an account at <https://pypi.org/account/register/> and turn on
   two-factor authentication (PyPI requires it before you can upload).
2. Make an API token at <https://pypi.org/manage/account/token/>. For the very
   first upload the token must be scoped to "Entire account", because the
   project does not exist yet. After that, make a new token scoped to the
   `capturethenarrative` project and delete the account-wide one.
3. Put the token where `twine` finds it, in `~/.pypirc`:

   ```ini
   [pypi]
   username = __token__
   password = pypi-AgEIcHlwaS5vcmc...
   ```

   Or skip the file and let `twine` prompt you each time.
4. Install the two tools:

   ```bash
   pip install --upgrade build twine
   ```

## Each release

1. Bump the version in **two** places so they agree: `version` in
   `pyproject.toml` and `__version__` in `src/capturethenarrative/__init__.py`. PyPI never
   lets you re-upload a version, even after deleting it, so every upload needs a
   new number.
2. Build the wheel and the source distribution:

   ```bash
   rm -rf dist
   python3 -m build
   ```

   This leaves `dist/capturethenarrative-X.Y.Z-py3-none-any.whl` and
   `dist/capturethenarrative-X.Y.Z.tar.gz`.
3. Check the metadata and README render cleanly:

   ```bash
   twine check dist/*
   ```

4. Optionally try it on TestPyPI first (separate account and token at
   <https://test.pypi.org>):

   ```bash
   twine upload --repository testpypi dist/*
   pip install --index-url https://test.pypi.org/simple/ \
       --extra-index-url https://pypi.org/simple/ capturethenarrative
   ```

   The extra index is where `openai` comes from; TestPyPI does not carry it.

5. Upload for real:

   ```bash
   twine upload dist/*
   ```

6. Confirm it installs from a clean environment:

   ```bash
   python3 -m venv /tmp/ctn-check && /tmp/ctn-check/bin/pip install capturethenarrative
   /tmp/ctn-check/bin/python -c "import capturethenarrative as ctn; print(ctn.__version__)"
   ```

## If you use `uv` instead

```bash
uv build
uv publish --token pypi-AgEIcHlwaS5vcmc...
```

`uv publish` reads `UV_PUBLISH_TOKEN` from the environment as well.

## Notes

- `ctn` itself is taken on PyPI (an empty project registered by someone else),
  so the distribution and the import are both `capturethenarrative`. The docs
  steer users to `import capturethenarrative as ctn`.
- `README.md` in this directory is the PyPI project page, so keep it current.
- The only dependency is `openai`, for `ctn.competitorllm`. The wheel is pure
  Python, so one file serves every platform.
