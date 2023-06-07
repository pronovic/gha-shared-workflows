# Developer Notes

Releasing is a totally manual process:

```
$ git checkout master && git pull
$ git tag -a -m "Release v3.1.1" v3.1.1
$ git tag -f -a -m "Release v3.1.1" v3
$ git push -f --tags
```

Once that's done, find the [new tag](https://github.com/pronovic/gha-shared-workflows/tags) and create a GitHub release from it.

