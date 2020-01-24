# pipenv2tools

Using Pipenv? Don't want to? Lets fix that.

## Why pip-tools?

* pip-tools emits a `pip`-readable file so does not need to be installed in CI
* pip-tools locks *much* faster
* Pipenv manages virtual environments
    * ...and I prefer pyenv
    * ...in a tool-unfriendly way
* Pipenv hasn't had a release in over a year as of writing (all of 2019)

## Quickstart

Get
```
curl -sSLO https://raw.githubusercontent.com/nicktimko/pipenv2tools/master/pipenv2tools
chmod +x pipenv2tools
./pipenv2tools --help
```

Run
```
# $ ls Pipfile*
# Pipfile   Pipfile.lock
./pipenv2tools
# $ ls Pipfile* requirements*
# Pipfile          Pipfile.lock     requirements.in  requirements.txt
# $ rm Pipfile Pipfile.lock
```

## What's this do?

Coverts a `Pipfile[.lock]` to a pip-tools `requirements.(in|txt)` file. While it's only slightly annoying to convert the `Pipfile` to a `requirements.in` (which we do) manually, the primary goal is to convert the *locked* file with all the versions and hashes, without *updating* any of them. This will allow bisection of problems to the conversion from Pipenv to pip-tools without confounding from countless updates to all the libraries used within.

To validate/clean up the new file, the script attempts to run `pip-compile` (if `-G` is not given). If it can't be found, manually run the command below which will use the lock file and inspect what needs to be updated (hopefully nothing), and should just change the "# via ?" lines and maybe re-order some packages.

```bash
$ pip-compile --quiet --generate-hashes --output-file=requirements.txt requirements.in
```

## Limitations

To fix...

* Doesn't split out dev requirements nicely, and can either dump them together in the `requirements.in` by default, or omit them entirely with `-D`
* Poor error handling when executables (`pip-compile`) can't be found
* Not `pip install`able itself

## Extras

### Makefile targets

If you have a `Makefile`, here are some targets you may wish to add:

```makefile
requirements.txt: requirements.in
    pip-compile \
        --quiet \
        --generate-hashes \
        --output-file=requirements.txt \
        requirements.in

# optional
venv_dir := ".venv"
venv_bin := "${venv_dir}/bin"
venv_python := "${venv_bin}/python3"
venv: requirements.txt
    rm -rf ${venv_dir}
    python3 -m venv ${venv_dir}
    ${venv_python} -m pip install \
        --requirement requirements.txt \
        --no-deps
    echo '# target sentinel' > $@
    ${venv_python} -m pip freeze >> $@
```

### Docker

Instead of installing Pipenv in your Docker container to simply read/install the `Pipfile[.lock]`, you can use pure `pip`.

```dockerfile
COPY requirements.txt /tmp/requirements.txt
RUN set -o nounset -o errexit -o xtrace -o verbose \
    ######
    # install any build dependencies here if you need to make packages
    # containing C-extensions
    && echo "apk add --no-cache --virtual .pip-install-deps <pkgs> ?" \
    && echo "apt install?"
    ######
    && VENV_PATH="/your/venv/path" \
    && python3 -m venv ${VENV_PATH} \
    && ${VENV_PATH}/bin/python3 -m pip install \
        --requirement /tmp/requirements.txt \
        # do not search for and install deps; these should be identified by
        # pip-compile converting requirements.in -> requirements.txt
        --no-deps \
        # (optional) reduce noise
        --disable-pip-version-check \
        # (optional) reduce layer size
        --no-cache \
    # (optional) uninstall the build deps
    && echo "apk del -q --purge .pip-install-deps?" \
    && echo "apt remove ...?"
```
