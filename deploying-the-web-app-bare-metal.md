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
* You have a valid GitHub Personal Access token, it does not need to have any permission, it's just used for increasing rate limit when we scrap information on the softwares like the latest version published. &#x20;

### Steps to be performet only once

{% code title="ssh code.gouv.fr" %}
```bash

sudo apt-get update

# Install node 18: https://github.com/nodesource/distributions#using-debian-as-root
# Other node version will do just fine
sudo su
curl -fsSL https://deb.nodesource.com/setup_18.x | bash - && apt-get install -y nodejs
exit

sudo npm install -g yarn
sudo npm install -g serve

mkdir ~/sill
cd sill

sudo apt-get install git
git clone https://github.com/codegouvfr/sill-api
git clone https://github.com/codegouvfr/sill

# Don't forget to replace by the correct values!
cat << EOF >> ~/.bashrc
export GITHUB_TOKEN=ghp_xxxxxx
export KEYCLOAK_CODEGOUV_ADMIN_PASSWORD=xxxxxx
export SSH_PRIVATE_KEY_FOR_GIT_NAME="id_ed25519"
export SSH_PRIVATE_KEY_FOR_GIT="-----BEGIN OPENSSH PRIVATE KEY-----\nxxxx\nxxxx\nxxxx\nAxxxx\nxxxx\n-----END OPENSSH PRIVATE KEY-----\n"
export GITHUB_SILL_WEBHOOK_SECRET=xxxxxxxx
EOF

source ~/.bashrc
```
{% endcode %}

Edit the SILL api configuration:

{% code title="~/sill-api/.env.local.sh" %}
```diff
#!/bin/bash

export CONFIGURATION=$(cat << EOF
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
  "port": 3049
- "isDevEnvironnement": true
+ "isDevEnvironnement": false
}
EOF
) 
$@
```
{% endcode %}

### Steps to be performed to put in production the latest version

{% code title="ssh code.gouv.fr" %}
```bash
cd ~/sill/sill-api
git fetch --tags
LATEST_TAG=$(git describe --tags `git rev-list --tags --max-count=1`)
git checkout $LATEST_TAG
yarn
yarn build

cd ~/sill/sill-web
git fetch --tags
LATEST_TAG=$(git describe --tags `git rev-list --tags --max-count=1`)
git checkout $LATEST_TAG
yarn
yarn build
```
{% endcode %}

### Start the web app

{% code title="ssh code.gouv.fr" %}
```bash
screen -S sill-api
source ~/.bashrc
cd ~/sill/sill-api
yarn start
# <CTRL>+A to exit the screen session, it can be restores with 'screen -r sill-api'
```
{% endcode %}

{% code title="ssh code.gouv.fr" %}
```bash
screen -S sill-web
cd ~/sill/sill-web
serve -p 4048 -s build
# <CTRL>+A to exit the screen session, it can be restores with 'screen -r sill-web'
```
{% endcode %}
