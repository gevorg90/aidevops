# Jenkins pnpm private package authentication

## Symptom

`pnpm install` fails with:

ERR_PNPM_TARBALL_HTTP_STATUS 404

for a private package such as:

@renderforest/...

## Important interpretation

A 404 from a private npm registry does not always mean the package is missing.

In Jenkins, first verify whether npm authentication is available inside the actual build environment/container.

## Troubleshooting

1. Inspect the Jenkins build command.
2. Check whether `.npmrc` is mounted into the container.
3. Check which user/home directory pnpm is using.
4. Verify the registry configured for the private scope.
5. Verify that the npm token/credential is available to that user.
6. Compare with a working Jenkins job if available.
7. Avoid printing the token itself.

## Learned pattern

When a private package works elsewhere but Jenkins gets a 404 during pnpm install, suspect registry authentication before assuming the package/version does not exist.

For internal `video-templates-ssr-1.0.N-development.tgz` dependencies, packages
may actually have been deleted from development Nexus storage. After confirming
the cause, rebuild the matching `video-templates-ssr/development-N` producer
before retrying the consumer. See
[missing development SSR package recovery](missing-development-video-templates-ssr-package.md).
