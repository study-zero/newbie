# newbie study docs
## sources
```
sphinx/source
```

## build
```sh
rye sync
make clean && make html
```

<!--
```sh
rye sync
rye run sphinx-build --version
rye run sphinx-build -M html docs/source/ docs/build/
```
-->

## serve locally(using output)
```sh
python -m http.server -d docs 4001
# use your own port
```

## contribution
### add sidebar contents
```sh
vi sphinx/source/index.rst
# append some toctree
```

### add posts on kuar
> A post must have a title(h1)

```sh
vi sphinx/source/kuar/---.md
# vi sphinx/source/kuar/---.rst
```

## tools
- https://rye.astral.sh/
- https://sphinx-rtd-theme.readthedocs.io/en/stable/installing.html

