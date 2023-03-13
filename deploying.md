# 🏁 Deploying the WebApp

This is a step by step guide for deploying sill.code.gouv.fr

### The Git based Database

#### Context

The data are stored in a [git based database](https://github.com/codegouvfr/sill-data-template).

* [the **main** branch](https://github.com/etalab/sill-data-template/tree/main): Each `.json` file represent a table like in any relational database.
* [the **compiled-data** branch](https://github.com/etalab/sill-data-template/tree/compiled-data): Contains two `.json` files. The first one, [compiledData\_private.json](https://github.com/codegouvfr/sill-data-template/blob/build/compiledData\_private.json) contains all the information that we have in [our database](https://github.com/etalab/sill-data-template/tree/main) and also the ones collected from different sources like Wikidata or LeComptoirDuLibre, compiled into a single file. This file contains emails of agents and should not be shared publicly. [compiledData\_public.json](https://github.com/codegouvfr/sill-data-template/blob/build/compiledData\_public.json) on the other end cand be shared, its the same file minus the personal infos about the agents. &#x20;

There is a bidirectional relationship betwen the Web App and the Data repo, when you update the data repo it updates the web App and the other way around is true as well.

The scrapping and update of the build branch is performed [once every four hour](https://github.com/codegouvfr/sill-api/blob/08c3c36b7e885fa1867e25fb30364f5a15c6c39f/src/core/usecases/readWriteSillData.ts#L155-L158) and [whenever there is a commit on the main branch](https://github.com/codegouvfr/sill-data-template/blob/41933d7ad1dc99ed1daa15fbebf9307b5b9c1ba5/.github/workflows/ci.yaml#L3-L5).

#### Instantiating a new database instance

* Click on the green button "_use this template_" and call it `sill-data`
* Check "_Include all branches_

####

### Provison a Kubernetes cluster



Currently we use the SSPCloud to deploy the SILL but if you have to deploy from scratch here is how provison and setup a Kubernetes cluser from a cloud provider. &#x20;

[Hashicorp](https://www.hashicorp.com/) maintains great tutorials for [terraforming](https://www.terraform.io/) Kubernetes clusters on [AWS](https://aws.amazon.com/what-is-aws/), [GCP](https://cloud.google.com/) or [Azure](https://acloudguru.com/videos/acg-fundamentals/what-is-microsoft-azure).

Pick one of the three and follow the guide.

You can stop after the [configure kubectl section](https://learn.hashicorp.com/tutorials/terraform/eks#configure-kubectl).

{% embed url="https://learn.hashicorp.com/tutorials/terraform/eks" %}

{% embed url="https://learn.hashicorp.com/tutorials/terraform/gke?in=terraform/kubernetes" %}

{% embed url="https://learn.hashicorp.com/tutorials/terraform/aks?in=terraform/kubernetes" %}

### Installing an ingress controller

Deploy an ingress controller on your cluster:

{% hint style="warning" %}
The following command is [for AWS](https://kubernetes.github.io/ingress-nginx/deploy/#aws).

For GCP use [this command](https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke).

For Azure use [this command](https://kubernetes.github.io/ingress-nginx/deploy/#azure).
{% endhint %}

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/aws/deploy.yaml
```

### Setting up the DSN

Now you need to get the external address of your cluster, run the command

```bash
kubectl get services -n ingress-nginx
```

and write down the `External IP` assigned to the `LoadBalancer`.

Depending on the cloud provider you are using it can be an IPv4, an IPv6 or a domain. On AWS for example, it will be a domain like **xxx.elb.eu-west-1.amazonaws.com**.

If you see `<pending>`, wait a few seconds and try again.

Once you have the address, create the following DNS records:

```dns-zone-file
sill.code.gouv.fr. CNAME xxx.elb.eu-west-1.amazonaws.com.
auth.code.gouv.net CNAME xxx.elb.eu-west-1.amazonaws.com.
```

If the address you got was an IPv4 (`x.x.x.x`), create a `A` record instead of a CNAME.

If the address you got was ans IPv6 (`y:y:y:y:y:y:y:y`), create a `AAAA` record.

### Creating a namespace for our apps

```bash
kubectl create namespace projet-codegouv
```

### SSL

{% embed url="https://github.com/codegouvfr/paris-sspcloud/tree/main/cert-manager" %}

### Installing Keycloak

```bash
helm repo add codecentric https://codecentric.github.io/helm-charts

POSTGRESQL_PASSWORD=xxxxx #Replace by a strong password, you will never need it.
# Credentials for logging to https://auth.code.gouv.fr/auth
KEYCLOAK_PASSWORD=yyyyyy 

cat << EOF > ./keycloak-values.yaml
image:
  tag: "18.0.2-legacy"
replicas: 1

nodeSelector:
  infra: "true"
tolerations:
  - key: "infra"
    operator: "Exists"

serviceAccount:
  create: true
  name: ""
  annotations: {}
  labels: {}
  imagePullSecrets: []


podSecurityContext:
  fsGroup: 1000


securityContext:
  runAsUser: 1000
  runAsNonRoot: true


extraInitContainers: |
  - name: realm-ext-provider
    image: curlimages/curl
    imagePullPolicy: IfNotPresent
    command:
      - sh
    args:
      - -c
      - |
        curl -L -f -S -o /extensions/keycloak-franceconnect-4.2.0.jar https://github.com/InseeFr/Keycloak-FranceConnect/releases/download/4.2.0/keycloak-franceconnect-4.2.0.jar
        # Here be carefull, the version number should match the version of sill-web in production.
        curl -L -f -S -o /extensions/sill-web.jar https://github.com/codegouvfr/sill-web/releases/download/v0.0.1/keycloak-theme.jar
    volumeMounts:
      - name: extensions
        mountPath: /extensions

extraVolumeMounts: |
  - name: extensions
    mountPath: /opt/jboss/keycloak/standalone/deployments

extraVolumes: |
  - name: extensions
    emptyDir: {}

# Additional environment variables for Keycloak
extraEnv: |
  - name: KEYCLOAK_USER
    value: admin
  - name: KEYCLOAK_PASSWORD
    value: $KEYCLOAK_PASSWORD
  - name: JGROUPS_DISCOVERY_PROTOCOL
    value: kubernetes.KUBE_PING
  - name: KUBERNETES_NAMESPACE
    valueFrom:
     fieldRef:
       apiVersion: v1
       fieldPath: metadata.namespace
  - name: KEYCLOAK_STATISTICS
    value: "true"
  - name: CACHE_OWNERS_COUNT
    value: "2"
  - name: CACHE_OWNERS_AUTH_SESSIONS_COUNT
    value: "2"
  - name: PROXY_ADDRESS_FORWARDING
    value: "true"
  - name: JAVA_OPTS
    value: >-
      -XX:+UseContainerSupport -XX:MaxRAMPercentage=50.0        -Djava.net.preferIPv4Stack=true        -Djboss.modules.system.pkgs=$JBOSS_MODULES_SYSTEM_PKGS   -Djava.awt.headless=true       -Dkeycloak.profile=preview

service:
  labels: {}
  type: ClusterIP
  loadBalancerIP: ""
  httpPort: 80
  httpNodePort: null
  httpsPort: 8443
  httpsNodePort: null
  httpManagementPort: 9990
  httpManagementNodePort: null
  extraPorts: []
  loadBalancerSourceRanges: []
  sessionAffinity: ""
  sessionAffinityConfig: {}

ingress:
  enabled: true
  servicePort: http
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffer-size: 128k

  labels: {}
  rules:
    - host: "tmp-auth-codegouv.lab.sspcloud.fr"
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - tmp-auth-codegouv.lab.sspcloud.fr
      secretName: sill-tls

networkPolicy:
  enabled: false
  labels: {}
  extraFrom: []

postgresql:
  enabled: true
  postgresqlUsername: keycloak
  postgresqlPassword: "$POSTGRESQL_PASSWORD"
  postgresqlDatabase: keycloak
  networkPolicy:
    enabled: false

EOF

helm install keycloak codecentric/keycloak -f keycloak-values.yaml -n projet-sill
```

You can now login to the **administration console** of **https://sill-auth.my-domain.net** and login using the credentials you have defined with `KEYCLOAK_USER` and `KEYCLOAK_PASSWORD`.

1. Create a realm called "codegouv" (or something else), go to **Realm settings**
   1. On the tab General
      1. _User Profile Enabled_: **On**
   2. On the tab **login**
      1. _User registration_: **On**
      2. _Forgot password_: **On**
      3. _Remember me_: **On**
      4. Verify email: **On**
   3. On the tab **email,** we give an example with \*\*\*\* [AWS SES](https://aws.amazon.com/ses/), if you don't have a SMTP server at hand you can skip this by going to **Authentication** (on the left panel) -> Tab **Required Actions** -> Uncheck "set as default action" **Verify Email**. Be aware that with email verification disable, anyone will be able to sign up to your service.
      1. _From_: **noreply@lab.my-domain.net**
      2. _Host_: **email-smtp.us-east-2.amazonaws.com**
      3. _Port_: **465**
      4. _Authentication_: **enabled**
      5. _Username_: **\*\*\*\*\*\*\*\*\*\*\*\*\*\***
      6. _Password_: **\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\***
      7. When clicking "save" you'll be asked for a test email, you have to provide one that correspond **to a pre-existing user** or you will get a silent error and the credentials won't be saved.
   4. On the tab Themes
      1. _Login theme_: **sill-web** (you can also select the login theme on a per client basis)
      2. _Email theme_: **sill-web**
      3. _Internationalization_: **Enabled**
      4. _Supported locales_: **en fr**
2. Create a client called "sill"
   1. _Root URL_: **https://sill.code.gouv.fr/**
   2. _Valid redirect URIs_: **https://sill.code.gouv.fr/\* ( and http://localhost\* for developpement)**
   3. _Web origins_: **\***
3. In **Authentication** (on the left panel) -> Tab **Required Actions** enable and set as default action **Therms and Conditions.**

Now you want to define a list of email domain allowed to register to your service.

Go to **Realm Settings** (on the left panel) -> Tab **User Profile** (this tab shows up only if User Profile is enabled in the General tab and you can enable user profile only if you have started Keycloak with `-Dkeycloak.profile=preview)` -> **JSON Editor**.

Now you can edit the file as suggested in the following DIFF snippet. Be mindful that in this example we only allow emails @gmail.com and @hotmail.com to register you want to edit that.

<pre class="language-diff"><code class="lang-diff"> {
   "attributes": [
     {
       "name": "email",
       "displayName": "${email}",
       "validations": {
         "email": {},
         "length": {
           "max": 255
         },
+       "pattern": {
<strong>+         "pattern": "^[^@]+@([^.]+\\.)*((gmail\\.com)|(hotmail\\.com))$"
</strong>+      }
       },
       "permissions": {
         "view": [],
         "edit": []
       },
       "selector": {
         "scopes": []
       }
     },
<strong>+    {
</strong>+      "selector": {
+        "scopes": []
+      },
+      "permissions": {
+        "view": [
+          "user",
+          "admin"
+        ],
+        "edit": [
+          "user",
+          "admin"
+        ]
+      },
+      "name": "agencyName",
+      "displayName": "${agencyName}",
+      "validations": {},
+      "required": {
+        "roles": [
+          "user"
+        ],
+        "scopes": []
+      }
+   }
   ]
 }
</code></pre>

Finally you need to create mapper so that agencyName appears in the JWT.

Go to clients -> sill -> Mappers

* Name: `agency name`
* Mapper type: `User attribute`
* User attribute: `Agency Name`
* Token claim name: `agency_name`
* Claim JSON type: `string`

#### Enabeling AgentConnect

<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

To enable agent connect you need to use [this extention](https://github.com/InseeFr/Keycloak-FranceConnect#agent-connect) (I's already loaded in your Keycloak if you look carefully in your `keycloak-values.yaml` file. )

Follow the instructions in the readme of [InseeFr/Keycloak-FranceConnect](https://github.com/InseeFr/Keycloak-FranceConnect#agent-connect).

Theses are the information that you'll need to give to the France Connect team to receive your credentials:

```
Nom du fournisseur de service: "Socle interministériel de logiciels libres" ( ou SILL pour les intimes )
URL de redirection de connexion: https://sill-auth.my-domain.net/auth/realms/etalab/broker/agentconnect/endpoint
URL de redirection de déconnexion: https://sill-auth.my-domain.net/auth/realms/etalab/broker/agentconnect/endpoint/logout_response
URL du bouton affichant AgentConnect: Le bouton AgentConnect sera affiché sur la page le login, on accède a la page de login en cliquant sur "Connexion" en haut à droite de l'écran sur https://sill.etalab.gouv.fr/
Scopes désirés: given_name usual_name email organizational_unit
Algorithme de la signature des userinfos: ES256
Algorithme de la signature de l'id token: ES256
```

You'll also need to create a mapper for organizational\_unit -> agencyName.

To do so, go to Identity Providers -> Agent Connect -> Mappers -> create

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

### Instantiating the web app

First of all you need to enable SSH autentication via private/public key on GitHub (or whatever platfrom you're using):

* Generate a priv/pub key if you don't have one already: `ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519 -C "john@example.com"`
* Got to your global GitHub setting, then SSH and GPG Keys, new SSH Key and pass the content of `~/.ssh/id_ed25519.pub`.

helm repo add etalab https://etalab.github.io/helm-charts

```bash
SSH_PRIVATE_KEY_NAME=id_ed25519 # For example, generated earlyer
SSH_PRIVATE_KEY="-----BEGIN OPENSSH PRIVATE KEY-----\nxxxx\nxxxx\nxxxx\nAxxxx\nxxxx\n-----END OPENSSH PRIVATE KEY-----\n"
DATA_REPO_SSH_URL="git@github.com:codegouvfr/sill-data.git" # Replace by the repo you created earlyer
KEYCLOAK_PASSWORD=yyyyyy # Make sure it's the same that the one you defined earlyer
WEBHOOK_SECRET=dSdSPxxxxxx # (optional) Some random caracters 
GITHUB_TOKEN=ghp_xxxxxx # (optional) A token, just for the GitHub API rate limit.  

cat << EOF > ./sill-values.yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: sill.code.gouv.fr
  tls:
    - hosts:
        - sill.code.gouv.fr
web:
  replicaCount: 2
  image:
    version: 0.0.1
  nodeSelector:
    infra: "true"
  tolerations:
    - key: "infra"
      operator: "Exists"
  
api:
  replicaCount: 1
  image:
    version: 0.0.1
  nodeSelector:
    infra: "true"
  tolerations:
    - key: "infra"
      operator: "Exists"
  env:
    CONFIGURATION: |
      {
        "keycloakParams": {
          "url": "https://auth.code.gouv.fr/auth",
          "realm": "codegouv",
          "clientId": "sill",
          "termsOfServices": "https://sill.code.gouv.fr/tos_fr.md",
          "adminPassword": "$KEYCLOAK_PASSWORD",
          "agencyNameAttributeName:": "agencyName"
        },
        "jwtClaimByUserKey": {
          "id": "sub",
          "email": "email",
          "agencyName": "agency_name",
          "locale": "locale"
        },
        "dataRepoSshUrl": "git@github.com:codegouvfr/sill-data.git",
        "sshPrivateKeyForGitName": "id_ed25519",
        "sshPrivateKeyForGit": "$SSH_PRIVATE_KEY",
        "githubWebhookSecret": "$WEBHOOK_SECRET",
        "githubPersonalAccessToken": "$GITHUB_TOKEN",
      }
EOF

helm install sill codegouvfr/sill -f sill-values.yaml -n projet-sill
```

### Enabling web hooks

By default the web app periodically checks the data repo for update.

If you want, and if you data repo is hosted on GitHub you can enable a Webhook that will ping the web app whenever there is an update.

<figure><img src=".gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

Type some random string as secret. You then need to provide it to `sill-api` so it know it can trust the ping to be genuin (you can do that later, for now just write down the secret).

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

sdf
