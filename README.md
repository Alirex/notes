# Notes

Different notes

| Note                                                                             | Language |
| -------------------------------------------------------------------------------- | -------- |
| [Dependencies](notes/dependencies/ukr.md)                                        | ukr      |
| [Ignore or keep IDE settings in the repo](notes/ignore_or_not_ide_config/ukr.md) | ukr      |
| [Code style, linters, formatters, pre-commit](notes/linters_pre_commit/ukr.md)   | ukr      |
| [Use "i" as a delimiter](notes/use_i_as_a_delimiter/eng.md)                      | eng      |

Saved answers:

| Note                                                                           | Language |
| ------------------------------------------------------------------------------ | -------- |
| [Why "-1" means "not found"](saved/minus_one_as_negative_search_result/eng.md) | eng      |

---

## Dev

### Install prek for pre-commit hooks

Needed for automatic linting. Once per machine, either as a [uv](https://docs.astral.sh/uv/) tool:

```shell
uv tool install prek
```

or with the installer of the latest release:

```shell
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/j178/prek/releases/latest/download/prek-installer.sh | sh
prek --version
```

Later, `uv tool upgrade prek` or `prek self update` brings it up to date.

### Register pre-commit hooks

Once per clone, after cloning the repository:

```shell
prek install
```

or, if the clone already has pre-commit hooks from `pre-commit`:

```shell
prek install --overwrite
```

No need to repeat it after changing the hooks, a commit or a pull.

### Run pre-commit hooks

If needed, run them manually.

```shell
prek run --all-files
```

Useful after changing the hooks. Or just to check if everything is fine.
