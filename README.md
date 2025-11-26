# Notes

Different notes

---

## Dev

### Install prek for pre-commit hooks

```shell
if ! command -v prek &> /dev/null; then
    curl --proto '=https' --tlsv1.2 -LsSf https://github.com/j178/prek/releases/download/v0.2.19/prek-installer.sh | sh &&\
    prek self update
else
    prek self update
fi
```

Note: Run with self-update for installing the latest version of prek. Maybe they will provide a better script later.

### Register pre-commit hooks

```shell
prek install
```

or, if you have pre-commit hooks installed before prek:

```shell
prek install --overwrite
```

### Run pre-commit hooks

```shell
prek run --all-files
```
