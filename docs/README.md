# Monitoring Stack full documentation

## Requirements

 * Python >= 3.6
 * Virtualenv

## Project setup

#### Create virtualenv

```console
virtualenv -p python3 .venv
source .venv/bin/activate
```

> Another way to create this environment
> ```console
> python3 -m venv ./.venv
> source .venv/bin/activate
> ```

#### Install dependencies

```
pip install -r requirements.txt
```

#### Deploy in local environment

```
mkdocs serve
```

#### Build static site

```
mkdocs build --clean
```
