# 👩💻 Setting up a development environment

Makes sur to put the name of your SSH key and the private key (generated when you created the sill-data repo) in your `~/.bash_profile` example:

<pre class="language-bash"><code class="lang-bash">export SILL_KEYCLOAK_URL=https://auth.code.gouv.fr/auth
export SILL_KEYCLOAK_REALM=codegouv
export SILL_KEYCLOAK_CLIENT_ID=sill
<strong>export SILL_KEYCLOAK_ADMIN_PASSWORD=xxxxxx
</strong>export SILL_KEYCLOAK_ORGANIZATION_USER_PROFILE_ATTRIBUTE_NAME=agencyName
export SILL_README_URL=https://git.sr.ht/~codegouvfr/logiciels-libres/blob/main/sill.md
export SILL_TERMS_OF_SERVICE_URL=https://code.gouv.fr/sill/tos_fr.md
export SILL_JWT_ID=sub
export SILL_JWT_EMAIL=email
export SILL_JWT_ORGANIZATION=organization
export SILL_DATA_REPO_SSH_URL=git@github.com:codegouvfr/sill-data-test.git
<strong>export SILL_SSH_NAME=id_ed25xxxxx
</strong>export SILL_SSH_PRIVATE_KEY="-----BEGIN OPENSSH PRIVATE KEY-----\nxxxx\nxxxx\nxxxx\nAxxxx\nxxxx\n-----END OPENSSH PRIVATE KEY-----\n"
<strong>export SILL_GITHUB_TOKEN=ghp_xxxxxx
</strong><strong>export SILL_WEBHOOK_SECRET=xxxxxxx
</strong>export SILL_API_PORT=3084
export SILL_IS_DEV_ENVIRONNEMENT=true
</code></pre>

You'll need [Node](https://nodejs.org/) and [Yarn 1.x](https://classic.yarnpkg.com/lang/en/). (Find [here](https://docs.gitlanding.dev/#step-by-step-guide) instructions by OS on how to install them)

It is much easyer to navigate the code with VSCode (We recommend the free distribution [VSCodium](https://sill.code.gouv.fr/software?name=VSCodium)).

```bash
#Only the first time

mkdir ~/sill

cd ~/sill
git clone https://github.com/codegouvfr/sill-web
cd sill-web
yarn

cd ~/sill
git clone https://github.com/codegouvfr/sill-api
cd sill-api
yarn
yarn build
yarn link-in-web

# Everyday

cd ~/sill/sill-api
npx tsc -w

# Open a new terminal
cd ~/sill/sill-api
yarn start # Note, there is no hot reload.  

#Open a new terminal
cd ~/sill/sill-web
yarn start

#The app is running on http://localhost:3000
```

### Deploying changes in production

#### Frontend (sill-web)

Update [the package.json version number](https://github.com/codegouvfr/sill-web/blob/faeeb89792ee1174fd345717a94ca6677a2adb42/package.json#L4) and push.

#### Backend (sill-api)

Same, update [the package.json version number](https://github.com/codegouvfr/sill-api/blob/77703b6ec2874792ad7d858f29b53109ee590de1/package.json#L3) and push. Don't forget however [to wait](https://github.com/codegouvfr/sill-api/actions) for the latest version [to be published on NPM](https://www.npmjs.com/package/sill-api). And update the version [sill-web's package.json](https://github.com/codegouvfr/sill-web/blob/faeeb89792ee1174fd345717a94ca6677a2adb42/package.json#L48). (You'll need to update the package.lock as well by running `yarn` again, you can just run `yarn add @codegouvfr/sill`, it's faster).
