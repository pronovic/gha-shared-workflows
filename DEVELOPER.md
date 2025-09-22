# Developer Notes

Releasing is a totally manual process:

```
$ git checkout main && git pull
$ git tag -a -m "Release v10.0.0" v10.0.0
$ git tag -f -a -m "Release v10.0.0" v10
$ git push -f --tags && git push
```

Once that's done, find the [new tag](https://github.com/pronovic/gha-shared-workflows/tags) and create a GitHub release from it.

