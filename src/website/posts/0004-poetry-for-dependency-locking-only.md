---
title: Using Poetry exclusively for locking dependencies
date: 2023-11-19 21:41
---

> **UPDATE**: Poetry 1.8.0 has been released, which introduces the `non-package` mode, solving many of the pain points described in here. I'll keep the old information and include a correction afterwards, so we all can see in posperity the basic shit they figured out too fucking late.

The bare minimum of bullcrap you'll have to accept in order to use Poetry exclusively for locking dependencies. No virtualenvs, no build system, no magic fuckery. Just lock dependencies.

Some amount of bullcrap might disappear once [this issue](https://github.com/python-poetry/poetry/issues/1132) is resolved or potentially [this pull request](https://github.com/python-poetry/poetry/pull/8650) is merged. Subscribe to those threads.

> **UPDATE**: _Guess what._

First, let's install poetry with the official installer instead of using pipx. Why? Because the installer is many orders of magnitude simpler than pipx so you don't depend on a package manager to install another package manager. You're welcome.

```bash
curl -sSL https://install.python-poetry.org | python3 -
# Yes, I pipe curl to python. Email me so we can fist fight.
```

Poetry assumes you always want virtual envs, but you don't.

```bash
# Option A: Config setting that lives somewhere in your system and
# I won't be the one investigating where.
poetry config virtualenvs.create false

# Option B: Keep this environment variable with this value all the
# time, maybe in your dotfiles? idk.
export POETRY_VIRTUALENVS_CREATE=false
```

Poetry assumes you always want a full fledged Python package so you can publish it on PyPI and ashame your friends and colleagues, but you don't. Sadly, Poetry doesn't care about your feelings, so you'll need to add tons of unnecessary shit in your `pyproject.toml` anyway.

```toml
[tool.poetry]
name = "goatse"
version = "0.0.0"
description = ""
authors = ["Poor Bastard <iam@depressed.com>"]

[tool.poetry.dependencies]
python = "^3.11"

# Everything above this line is shit so Poetry doesn't complain.
# Your dependencies go here!
flask = { version = "2.3.3" }
```

> **UPDATE**: It got simpler after Poetry 1.8.0 folks! Rejoice in eventual sanity! Now it looks like this:

```toml
[tool.poetry]
package-mode = false

[tool.poetry.dependencies]
python = "^3.11"

# Everything above this line is shit so Poetry doesn't complain.
# Your dependencies go here!
flask = { version = "2.3.3" }
```

And now you can install your dependencies.

## Now with Docker baby

The pain isn't over.

```dockerfile
# Explicit version
ENV POETRY_VERSION 1.7.1
```

> **UPDATE**: :D

```dockerfile
# woosh ✨
ENV POETRY_VERSION 1.8.2
```

Changing `POETRY_HOME` isn't necessary, but looking into the documentation to know what's the actual place it puts poetry by default so you can extend `PATH` _is_ necessary. Now making `POETRY_HOME` explicit doesn't sound that bad huh.

```dockerfile
ENV POETRY_HOME /poetry
ENV PATH="/poetry/bin:${PATH}"
```

Probably you should download the install script into source control and copy it into the container instead. This is an example okay?

```dockerfile
RUN curl -sSL https://install.python-poetry.org | python -

RUN poetry config virtualenvs.create false
# Or...
ENV POETRY_VIRTUALENVS_CREATE false
```

It's good practice to copy first the files containing the dependencies, install them, and _then_ copy the rest of the sources, so rebuilding because the code changed doesn't take forever. Optimize your Docker cache folks.

```dockerfile
COPY ./pyproject.toml /app/pyproject.toml
COPY ./poetry.lock /app/poetry.lock
```

We're inside a Docker build, no stdin can help you here, so use `--no-interaction` in `poetry install`.

Poetry assumes you always want a full fledged Python package, _again_, and tries to build stuff which involves verifying that your code is actually where it's supposed to be. If your project is called `goatse`, will look for the `goatse` folder and fail, because it isn't there yet, because we're trying to optimize the Docker build. Add `--no-root` and dodge the issue.

```dockerfile
RUN cd /app && poetry install --no-root --no-interaction
```

> **UPDATE**: The whole paragraph above doesn't apply anymore because there's no package to manage, finally.

```dockerfile
RUN cd /app && poetry install --no-interaction
```

And we're done.
