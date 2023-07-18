# ⚒ Deploying the web App (Bare metal)

This are the instructions for deploying the web app "bare metal", without Helm, Kubernetes and Docker, just running the web app on a Debian machine.  \
\
Unlike the guide for deploying on Kubernetes where we deploy the SILL at the root of a domain, `sill.code.gouv.fr`, in this guide we deploy under a sub path: `code.gouv.fr/sill`.

In this section we make the following assumption: &#x20;

*   You have a Keycloak server running somewhere. You have

    * It's URL. E.g: `https://auth.code.gouv.fr/auth`)&#x20;
    * The password of the 'admin' user.&#x20;

    Refer to [this section](deploying.md#installing-keycloak) if you're not there yet.
*   You have a sill-data git repository setup. You have:

    * It's address, E.g: `git@github.com:codegouvfr/sill-data.git`
    * The credential of a user that have push permition on this repo. E.g:&#x20;
      * name: `id_edxxxxxx`
      * private key: `-----BEGIN OPENSSH PRIVATE KEY-----\nxxxx\nxxxx\nxxxx\nAxxxx\nxxxx\n-----END OPENSSH PRIVATE KEY-----\n`
    * (Optional): The web hook secret that enables to subscribe to to update event.

    If you're not there yet you can refere to [the instructions related to seting up the git database](deploying.md#the-git-based-database) through out this guide.
*   You own the domain code.gouv.fr and you have registered the DNS recod:

    `code.gouv.fr A <The public IPv4 of your Debian server>`



```bash
ssh code.gouv.fr

sudo apt-get update

# Install node 18: https://github.com/nodesource/distributions#using-debian-as-root
# Other node version will do just fine
sudo su
curl -fsSL https://deb.nodesource.com/setup_18.x | bash - && apt-get install -y nodejs
exit

sudo npm install -g yarn

mkdir ~/sill
cd sill

sudo apt-get install git
git clone https://github.com/codegouvfr/sill-api

cd sill-api

# Install dependencies
yarn

# Build the project
yarn build

vim .env.local.sh
```



<pre class="language-diff"><code class="lang-diff">#!/bin/bash

export CONFIGURATION=$(cat &#x3C;&#x3C; EOF
{
  "keycloakParams": {
    "url": "https://auth.code.gouv.fr/auth",
    "realm": "codegouv",
    "clientId": "sill",
    "adminPassword": "$KEYCLOAK_CODEGOUV_ADMIN_PASSWORD",
    "organizationUserProfileAttributeName": "agencyName"
  },
  "readmeUrl": "https://git.sr.ht/~codegouvfr/logiciels-libres/blob/main/sill.md",
  "termsOfServiceUrl": "https://code.gouv.fr/sill/tos_fr.md",
  "jwtClaimByUserKey": {
    "id": "sub",
    "email": "email",
    "organization": "organization"
  },
- "dataRepoSshUrl": "git@github.com:codegouvfr/sill-data-test.git",
+ "dataRepoSshUrl": "git@github.com:codegouvfr/sill-test.git",
  "sshPrivateKeyForGitName": "$SSH_PRIVATE_KEY_FOR_GIT_NAME",
  "sshPrivateKeyForGit": "$SSH_PRIVATE_KEY_FOR_GIT",
  "githubPersonalAccessTokenForApiRateLimit": "$GITHUB_TOKEN",
  "githubWebhookSecret": "$GITHUB_SILL_WEBHOOK_SECRET",
<strong>- "port": 8080,
</strong><strong>+ "port": 3049
</strong>- "isDevEnvironnement": true
+ "isDevEnvironnement": false
}
EOF
) 
$@
</code></pre>

