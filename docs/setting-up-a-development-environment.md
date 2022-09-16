# 👩💻 Setting up a development environment

Before anything, you'll need a sill-data repository that acts as a database of sort. Find the instructions [here](<README (1).md#the-data-git-repository>).

Once you have it you should edit the .env.local.sh of both etalab/sill-web and etalab/sill-api .env.local.sh to adapt with your environement.

* etalab/sill-web: [.env.local.sh](https://github.com/etalab/sill-web/blob/main/.env.local.sh) and [here is the shema](https://github.com/etalab/sill-web/blob/5dfd9535e777400e20dcd45dd3baf6173273e620/src/configuration.ts#L14-L65).
* etalab/sill-api: [.env.local.sh](https://github.com/etalab/sill-api/blob/main/.env.local.sh) and [here is the shema](https://github.com/etalab/sill-api/blob/main/src/server/configuration.ts).

If you don't have a Keycloak server you can disable authentication by removing the keycloakParams from [etalab/sill-api/.env.local.sh](https://github.com/etalab/sill-api/blob/2a6016fbd408bded77acf3938e3c74cea76c53aa/.env.local.sh#L5-L11) if you have one you must have [an environement variable](https://github.com/etalab/sill-api/blob/2a6016fbd408bded77acf3938e3c74cea76c53aa/.env.local.sh#L10) that holds the password of the "admin" user.

Makes sur to put the name of your SSH key and the private key (generated when you created the sill-data repo) in your `~/.bash_profile` example:

```bash
export SSH_PRIVATE_KEY_FOR_GIT_NAME="id_ed25519"
export SSH_PRIVATE_KEY_FOR_GIT="-----BEGIN OPENSSH PRIVATE KEY-----\nXXX\nXXX\nXXX\nXXX\nXXX\n-----END OPENSSH PRIVATE KEY-----\n"
```

You'll need [Node](https://nodejs.org/) and [Yarn 1.x](https://classic.yarnpkg.com/lang/en/). (Find [here](https://docs.gitlanding.dev/#step-by-step-guide) instructions by OS on how to install them)

It is much more easy to navigate the code with [VSCode](https://code.visualstudio.com/).

```bash
cd ~/github
git clone https://github.com/etalab/sill-api
cd sill-api
yarn
# Each time you change something on the server
# you should re-run the two following command.
yarn build # 'npx tsc -w' for watch mode compilation
yarn dev #Start the server on port 8080

# Open a new terminal

cd ~/github
git clone https://github.com/etalab/sill-web
cd sill-web
yarn
yarn link_api #This make sure that you use your local version of "sill-web" and not the latest NPM release.
yarn start #The app is running on http://localhost:3000
```

### Linking inhouse dependencies

`sill-web` leverage a lot of inhouse NPM module, namely:

* [sill-api](https://www.npmjs.com/package/sill-api)
* [tss-react](https://tss-react.dev)
* [powerhooks](https://powerhooks.dev)
* [tsafe](https://tsafe.dev)
* [keycloakify](https://keycloakify.dev)
* [evt](https://evt.land)
* [i18nifty](https://github.com/garronej/i18nifty)
* [onyxia-ui](https://github.com/InseeFrLab/onyxia-ui)
* [cra-env](https://github.com/etalab/cra-env)

You may want to debug theses lib directly in `sill-web`. To do so simply do (we make an example with `sill-api` but replace keycloakify by any of the above)

```bash
cd ~/github
git clone https://github.com/etalab/sill-api
cd sill-api
yarn
yarn build
npx tsc -w # This will start compilation in watch mode on Keycloakify

# Open a new terminal

cd ~/github/sill-web
ts-node --skip-project src/bin/link_inhouse_deps.ts sill-api #You can link more than just one lib example: ts-node --skip-project src/bin/link_inhouse_deps.ts sill-api keycloakify onyxia-ui
yarn start
# Now, you can change Keycloakify and you will see the changes live in https://localhost:3000
```

### Deploying changes in production

#### Frontend (sill-web)

Update [the package.json version number](https://github.com/etalab/sill-web/blob/faeeb89792ee1174fd345717a94ca6677a2adb42/package.json#L4) and push.

#### Backend (sill-api)

Same, update [the package.json version number](https://github.com/etalab/sill-api/blob/77703b6ec2874792ad7d858f29b53109ee590de1/package.json#L3) and push. Don't forget however [to wait](https://github.com/etalab/sill-api/actions) for the latest version [to be published on NPM](https://www.npmjs.com/package/sill-api). And update the version [sill-web's package.json](https://github.com/etalab/sill-web/blob/faeeb89792ee1174fd345717a94ca6677a2adb42/package.json#L48). (You'll need to update the package.lock as well by running `yarn` again, you can just run `yarn add sill-api`, it's faster).

### Starting with Docker

This is how to build and start the Docker image of [etalab/sill-api](https://github.com/etalab/sill-api), you can adapt theses commands with [etalab/sill-web](https://github.com/etalab/sill-web).

```bash
# In your ~/.bash_profile
# export KEYCLOAK_ETALAB_ADMIN_PASSWORD=xxx
# export SSH_PRIVATE_KEY_FOR_GIT_NAME="id_ed25519" (for example)
# export SSH_PRIVATE_KEY_FOR_GIT="-----BEGIN OPENSSH PRIVATE KEY-----\nxxx\nxxx\nxxx\n-----END OPENSSH PRIVATE KEY-----\n"

CONFIGURATION=$(cat << EOF
{
  "keycloakParams": {
    "url": "https://sill-auth.etalab.gouv.fr/auth",
    "realm": "etalab",
    "clientId": "sill",
    "termsOfServices": "https://sill.etalab.gouv.fr/tos_fr.md",
    "adminPassword": "$KEYCLOAK_ETALAB_ADMIN_PASSWORD"
  },
    "jwtClaims": {
    "id": "sub",
    "email": "email",
    "agencyName": "agency_name",
    "locale": "locale"
  },
  "dataRepoSshUrl": "git@github.com:etalab/sill-data-test.git",
  "sshPrivateKeyForGitName": "$SSH_PRIVATE_KEY_FOR_GIT_NAME",
  "sshPrivateKeyForGit": "$SSH_PRIVATE_KEY_FOR_GIT"
}
EOF
)

docker build -t etalab/sill-api:main .
docker run -it -p 8080:8080 --env CONFIGURATION="$CONFIGURATION" etalab/sill-api:mainsh
```
