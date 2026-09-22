A demonstration of a DDEV DB image with data baked in. This is great for sites that have very large databases. The image may be used in CI for testing sites with data, and during development by developers and agents. Having the data baked in encures that each worktree has a DB ready to go on each `ddev start`. Notable files

- DB image building via daily Github Actions workflow: [database.yml](.github/workflows/database.yml), [Dockerfile](.github/database-ddev/Dockerfile)
- Convenience commands for developers: [pulldb](.ddev/commands/web/pulldb), [resetdb](.ddev/commands/web/resetdb)
- DDEV config that uses the built image: [config.yaml](.ddev/config.yaml)

Note that the [database.yml](.github/workflows/database.yml) in this repo fetches its DB dump from a Github release. A real site would fetch it from its hosting provider, S3, etc.
