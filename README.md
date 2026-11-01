# template

The starting point for new `ungood` repos. It passes the
[standards](https://github.com/ungood/standards) from its first commit.

To start a repo from it:

```sh
gh repo create ungood/<name> --public --template ungood/template --clone
```

Then:

1. Replace this README with one for the new repo.
2. Add the repo's tools to `mise.toml`, and its linters and formatters to
   `hk.pkl`.
3. Replace the `test` target once there are tests.
4. Run `mise run check`.

Changes to this template reach new repos only. Existing repos get changes to
the standards through Renovate.
