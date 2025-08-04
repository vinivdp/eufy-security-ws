# Deployment instructions

## eufy-security-client

[Instructions and dependency tree](https://github.com/bropat/eufy-security-client/tree/develop/docs/deployment.md).

## eufy-security-ws

How to deploy a new version of eufy-security-ws:

1. Update all the npm dependencies and bring in the [eufy-security-client npm package](https://www.npmjs.com/package/eufy-security-client) latest published version.
2. Review and merge into the [develop](https://github.com/bropat/eufy-security-ws/tree/develop) branch the PRs that should be included in the next release.
3. Merge everything from `develop` into [master](https://github.com/bropat/eufy-security-ws/tree/master).
4. Publish a new [release and tag](https://github.com/bropat/eufy-security-ws/releases/new) out of the latest changes merged into `master`.
5. Using that new release from `master`, publish a new [eufy-security-ws npm package version](https://www.npmjs.com/package/eufy-security-ws).

## hassio-eufy-security-ws

[Instructions](https://github.com/bropat/hassio-eufy-security-ws/tree/develop/eufy-security-ws/deployment.md).
