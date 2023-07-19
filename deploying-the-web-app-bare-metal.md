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

### Initial setup

Here are steps that are to be perfomed only once, for setting up everything.

<pre class="language-bash" data-title="ssh code.gouv.fr"><code class="lang-bash">
sudo apt-get update

# Install node 18: https://github.com/nodesource/distributions#using-debian-as-root
# Other node version will do just fine
sudo su
curl -fsSL https://deb.nodesource.com/setup_18.x | bash - &#x26;&#x26; apt-get install -y nodejs
exit

sudo npm install -g yarn
sudo npm install -g serve

mkdir ~/sill
cd sill

sudo apt-get install git
git clone https://github.com/codegouvfr/sill-api
git clone https://github.com/codegouvfr/sill-web

# Don't forget to replace by the correct values! 
# (or at least everywhere there is "xxxx")
cat &#x3C;&#x3C; EOF > sill-api/.env.local
export SILL_KEYCLOAK_URL=https://auth.code.gouv.fr/auth
export SILL_KEYCLOAK_REALM=codegouv
export SILL_KEYCLOAK_CLIENT_ID=sill
<strong>export SILL_KEYCLOAK_ADMIN_PASSWORD=xxxxxx
</strong>export SILL_KEYCLOAK_ORGANIZATION_USER_PROFILE_ATTRIBUTE_NAME=agencyName
export SILL_README_URL=https://git.sr.ht/~codegouvfr/logiciels-libres/blob/main/sill.md
export SILL_TERMS_OF_SERVICE_URL=https://code.gouv.fr/sill/tos_fr.md
export SILL_JWT_ID=sub
export SILL_JWT_EMAIL=email
export SILL_JWT_ORGANIZATION=organization
export SILL_DATA_REPO_SSH_URL=git@github.com:codegouvfr/sill-data.git
<strong>export SILL_SSH_NAME=id_ed25xxxxx
</strong><strong>export SILL_SSH_PRIVATE_KEY="-----BEGIN OPENSSH PRIVATE KEY-----\nxxxx\nxxxx\nxxxx\nAxxxx\nxxxx\n-----END OPENSSH PRIVATE KEY-----\n"
</strong><strong>export SILL_GITHUB_TOKEN=ghp_xxxxxx
</strong><strong>export SILL_WEBHOOK_SECRET=xxxxxxx
</strong>export SILL_API_PORT=3084
export SILL_IS_DEV_ENVIRONNEMENT=false
EOF
</code></pre>

#### Ngnix configuration

We assume you already have NGNIX installed and you have a TLS certificate for the domain code.gouv.fr under `/etc/letsencrypt/live/code.gouv.fr/{fullchan,privkey}.pem`

```nginx
# /etc/nginx/sites-available/code.gouv.fr

server {
    listen 80;
    server_name code.gouv.fr;
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name code.gouv.fr;

    ssl_certificate /etc/letsencrypt/live/code.gouv.fr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/code.gouv.fr/privkey.pem;

    location ~ ^/sill/api {
        proxy_pass http://localhost:3084;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location ~ ^/sill {
        proxy_pass http://localhost:3083;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Update & build

This is the protocoll to follow whenever a new version of the sill have been published.  \
We checkout the latest tag, install the new dependencies (if any) and re-build.

Don't forget to re-lauch the app afterward. &#x20;

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

### Start

These are the step to start both the frontend and the backend respectively on port 3048 and 3049 .

{% code title="ssh code.gouv.fr" %}
```bash
screen -S sill-api
cd ~/sill/sill-api
source .env.local
yarn start
# <CTRL>+A to exit the screen session, it can be restores with 'screen -r sill-api'
```
{% endcode %}

{% code title="ssh code.gouv.fr" %}
```bash
screen -S sill-web
cd ~/sill/sill-web
serve -p 3083 -s build
# <CTRL>+A to exit the screen session, it can be restores with 'screen -r sill-web'
```
{% endcode %}
