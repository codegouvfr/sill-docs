# 🏁 Deploying

This is a step by step guide that will assist you in installing your own instance of [sill.etalab.gouv.fr](https://sill.etalab.gouv.fr). &#x20;

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

### The Data Git repository

#### Context

The data are stored in [a separate Git repo](https://github.com/etalab/sill-data-template) that serves as source of truth for the SILL Web Application, a database of sort. &#x20;

This repo hold JSON files that describes what [software](https://github.com/etalab/sill-data-template/blob/main/software.json) and [services](https://github.com/etalab/sill-data-template/blob/main/service.json) are in the SILL. &#x20;

There is a bidirectional relationship betwen the Web App and the Data repo, when you update the data repo it updates the web App and the other way is true as well. &#x20;

On the main branch of the Data repo are stored only the the informations about the software that are directly related with the SILL and can't be found elswhere. &#x20;

There is [an other branch, the `build` banch](https://github.com/etalab/sill-data-template/tree/build) that holds all the informations of the main branch plus informations that have been scrapped from multiple sources like [Wikidata.org](https://www.wikidata.org/wiki/Wikidata:Main\_Page) and [https://comptoir-du-libre.org](https://comptoir-du-libre.org/en/).&#x20;

The scrapping and update of the build branch is performed [once every four hour ](https://github.com/etalab/sill-data-template/blob/b2a763f73fb1e38833a709e7403f0c359ec711a9/.github/workflows/ci.yaml#L7)and [whenever there is a commit on the main branch](https://github.com/etalab/sill-data-template/blob/b2a763f73fb1e38833a709e7403f0c359ec711a9/.github/workflows/ci.yaml#L2-L5). &#x20;

The data repo can be private or public and does not need to be hosted on GitHub. &#x20;

#### Instantiating your Data Git Repository

First of all you need to enable SSH autentication via private/public key on GitHub (or whatever platfrom you're using). &#x20;

* Generate a priv/pub key if you don't have one already: `ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519 -C "john@example.com"`
* Got to your global GitHub setting, then SSH and GPG Keys, new SSH Key and pass the content of `~/.ssh/id_ed25519.pub`. &#x20;

Now you want to start from [etalab/sill-data-template](https://github.com/etalab/sill-data-template)

* Click on the green button "_use this template_"
* Check "_Include all branches_"
* Navigate to the repo setting, security -> Secrets -> Actions, create two repository secrets:
  * `SSH_PRIVATE_KEY_NAME` whith content `id_ed25519` (or whatever you have)
  * `SSH_PRIVATE_KEY`_`NAME` and pass the content of_ `~/.ssh/id_ed25519` (it starts with `-----BEGIN OPENSSH PRIVATE KEY-----`)

Congratulation! 🥳 You now have a self managed repo. If you add a software to software.json [compiledData.json](https://github.com/etalab/sill-data-template/blob/build/compiledData.json) and [compiledData\_withoutReferents.json](https://github.com/etalab/sill-data-template/blob/build/compiledData\_withoutReferents.json) are going to be automatically updated.

{% hint style="warning" %}
Note that the CI is always using [the latest version of the scrapping script](https://github.com/etalab/sill-data-template/blob/b2a763f73fb1e38833a709e7403f0c359ec711a9/.github/workflows/ci.yaml#L13). You probably want to keep in sync with the version of etalab/sill-api you have in prod and (we depoly it later in this guide). &#x20;

Example:&#x20;

```diff
-npx -y -p sill-api@latest build-data
+npx -y -p sill-api@0.22.6 build-data
```
{% endhint %}

### Provison a Kubernetes cluster

First you'll need a Kubernetes cluster.  If you have one already you can skip this section.

{% tabs %}
{% tab title="Provisioning a cluster on AWS, GCP or Azure" %}
[Hashicorp](https://www.hashicorp.com/) maintains great tutorials for [terraforming](https://www.terraform.io/) Kubernetes clusters on [AWS](https://aws.amazon.com/what-is-aws/), [GCP](https://cloud.google.com/) or [Azure](https://acloudguru.com/videos/acg-fundamentals/what-is-microsoft-azure).&#x20;

Pick one of the three and follow the guide.&#x20;

You can stop after the [configure kubectl section](https://learn.hashicorp.com/tutorials/terraform/eks#configure-kubectl). &#x20;

{% embed url="https://learn.hashicorp.com/tutorials/terraform/eks" %}

{% embed url="https://learn.hashicorp.com/tutorials/terraform/gke?in=terraform/kubernetes" %}

{% embed url="https://learn.hashicorp.com/tutorials/terraform/aks?in=terraform/kubernetes" %}

### Ingress controller &#x20;

Deploy an ingress controller on your cluster:

{% hint style="warning" %}
The following command is [for AWS](https://kubernetes.github.io/ingress-nginx/deploy/#aws). &#x20;

For GCP use [this command](https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke).

For Azure use [this command](https://kubernetes.github.io/ingress-nginx/deploy/#azure).
{% endhint %}

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/aws/deploy.yaml
```

### DNS

Let's assume you own the domain name **my-domain.net**, for the rest of the guide you should replace **my-domain.net** by a domain you actually own. &#x20;

(In our case my-domain.net is etalab.gouv.fr)

Now you need to get the external address of your cluster, run the command&#x20;

```bash
kubectl get services -n ingress-nginx
```

and write down the `External IP` assigned to the `LoadBalancer`.&#x20;

Depending on the cloud provider you are using it can be an IPv4, an IPv6 or a domain. On AWS for example, it will be a domain like **xxx.elb.eu-west-1.amazonaws.com**. &#x20;

If you see `<pending>`, wait a few seconds and try again. &#x20;

Once you have the address, create the following DNS records:

```dns-zone-file
sill.my-domain.net CNAME xxx.elb.eu-west-1.amazonaws.com.
sill-auth.my-domain.net CNAME xxx.elb.eu-west-1.amazonaws.com.
sill-demo.my-domain.net CNAME xxx.elb.eu-west-1.amazonaws.com.
*.sill-tmp.my-domain.net  CNAME xxx.elb.eu-west-1.amazonaws.com. 
```

{% hint style="info" %}
Note that you can pick any subdomain you'd like in place of **sill**, **sill-auth** **sill-demo**, **sill-tmp**. &#x20;

If you do not plan to deploy the platform that enables to test the software in the browser you won't need the sill-demo and \*.sill-tmp record. &#x20;
{% endhint %}



If the address you got was an IPv4 (`x.x.x.x`), create a `A` record instead of a CNAME.

If the address you got was ans IPv6 (`y:y:y:y:y:y:y:y`), create a `AAAA` record.

* **https://sill.my-domain.net** will be the URL for [your instance of the SILL](https://sill.etalab.gouv.fr/).&#x20;
* **https://sill-auth.my-domain.net** will be the URL of [your Keycloak server](https://sill-auth.etalab.gouv.fr/auth/).
* **https://sill-demo.my-domain.net** will be the url [your Onyxia instance](https://sill-demo.etalab.gouv.fr/catalog/helm-charts-sill) for enabling users to test the software.
* **https://\*.sill-tmp.my-domain.net** will be the temporary test urls created by [Onyxia](https://www.onyxia.sh/). &#x20;

### SSL

In this section we will obtain a TLS certificate issued by [LetsEncrypt](https://letsencrypt.org/) using the [certbot](https://certbot.eff.org/) commend line tool then get our ingress controller to use it. &#x20;

If you are already familiar with `certbot` you're probably used to run it on a remote host via SSH. In this case you are expected to run it on your own machine, we'll use the DNS chalenge instead of the HTTP chalenge.

```bash
brew install certbot #On Mac, lookup how to install certbot for your OS

#Because we need a wildcard (*) certificate (for onyxia) we have to complete the DNS callange.  
sudo certbot certonly --manual --preferred-challenges dns

# When asked for the domains you wish to optains a certificate for enter:
#   sill.sill.ovh sill-auth.sill.ovh sill-demo.sill.ovh *.sill-tmp.sill.ovh
```

{% hint style="info" %}
The obtained certificate needs to be renewed every three month. &#x20;

To avoid the burden of having to remember to re-run the `certbot` command periodically you can setup [cert-manager](https://cert-manager.io/) and configure a [DNS01 challange provider](https://cert-manager.io/docs/configuration/acme/dns01/) on your cluster. You may need to delegate your DNS Servers to one of the supported [DNS service provider](https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers).

If you are not planing to deploy an Onyxia instance you do not need a wildcard (\*) certificate and thus, in place of DNS01 you can configure [the HTTP01 Ingress solver](https://cert-manager.io/docs/configuration/acme/http01/#configuring-the-http01-ingress-solver) which is much easier to configure. You can follow [this tutorial](https://www.youtube.com/watch?v=hoLUigg4V18).
{% endhint %}

Now we want to create a Kubernetes secret containing our newly obtained certificate: &#x20;

```bash
DOMAIN=my-domain.net
sudo kubectl create secret tls sill-tls \
    -n ingress-nginx \
    --key /etc/letsencrypt/live/sill.$DOMAIN/privkey.pem \
    --cert /etc/letsencrypt/live/sill.$DOMAIN/fullchain.pem
```

Lastly, we want to tell our ingress controller to use this TLS certificate, to do so run: &#x20;

```bash
kubectl edit deployment ingress-nginx-controller -n ingress-nginx
```

This command will open your configured text editor, go to line `56` and add: &#x20;

```yaml
        - --default-ssl-certificate=ingress-nginx/sill-tls
```

![](<.gitbook/assets/image (11).png>)
{% endtab %}

{% tab title="Test on your machine" %}
If you are on a Mac or Window computer you can install [Docker desktop](https://www.docker.com/products/docker-desktop/) then enable Kubernetes.

![Enable Kubernetes in Docker desktop](<.gitbook/assets/image (13).png>)

{% hint style="info" %}
Docker desktop isn't available on Linux, you can use [Kind](https://kind.sigs.k8s.io/) instead.
{% endhint %}

### Port Forwarding

You'll need to [forward the TCP ports 80 and 443 to your local machine](https://user-images.githubusercontent.com/6702424/174459930-23fb577c-11a2-49ef-a082-873f4139aca1.png).  It's done from the administration panel of your domestic internet Box. If you're on a corporate network, no luck for you I'm afraid.

### DNS

Let's assume you own the domain name **my-domain.net**, for the rest of the guide you should replace **my-domain.net** by a domain you actually own. &#x20;

(In our case my-domain.net is etalab.gouv.fr)

Get [your internet box routable IP](http://monip.org/) and create the following DNS records: &#x20;

```dns-zone-file
sill.my-domain.net A <YOUR IP>
sill-auth.my-domain.net A <YOUR IP>
sill-demo.my-domain.net A <YOUR IP>
*.sill-tmp.my-domain.net A <YOUR IP>
```

{% hint style="success" %}
If you have DDNS domain you can create `CNAMEs` instead example: &#x20;

```
sill.my-domain.net CNAME jhon-doe-home.ddns.net.
...
```
{% endhint %}

{% hint style="info" %}
Note that you can pick any subdomain you'd like in place of **sill**, **sill-auth** **sill-demo**, **sill-tmp**. &#x20;

If you do not plan to deploy the platform that enables to test the software in the browser you won't need the sill-demo and \*.sill-tmp record. &#x20;
{% endhint %}

* **https://sill.my-domain.net** will be the URL for [your instance of the SILL](https://sill.etalab.gouv.fr/).&#x20;
* **https://sill-auth.my-domain.net** will be the URL of [your Keycloak server](https://sill-auth.etalab.gouv.fr/auth/).
* **https://sill-demo.my-domain.net** will be the url [your Onyxia instance](https://sill-demo.etalab.gouv.fr/catalog/helm-charts-sill) for enabling users to test the software.
* **https://\*.sill-tmp.my-domain.net** will be the temporary url created by the Onyxia instance for testing the software. &#x20;

### SSL

In this section we will obtain a TLS certificate issued by [LetsEncrypt](https://letsencrypt.org/) using the [certbot](https://certbot.eff.org/) commend line tool. &#x20;

```bash
brew install certbot #On Mac, lookup how to install certbot for your OS

#Because we need a wildcard certificate we have to complete the DNS callange.  
sudo certbot certonly --manual --preferred-challenges dns

# When asked for the domains you wish to optains a certificate for enter:
#   #   sill.sill.ovh sill-auth.sill.ovh sill-demo.sill.ovh *.sill-tmp.sill.ov
```

{% hint style="info" %}
The obtained certificate needs to be renewed every three month. &#x20;

To avoid the burden of having to remember to re-run the `certbot` command periodically you can setup [cert-manager](https://cert-manager.io/) and configure a [DNS01 challange provider](https://cert-manager.io/docs/configuration/acme/dns01/) on your cluster. You may need to delegate your DNS Servers to one of the supported [DNS service provider](https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers).

If you are not planing to deploy an Onyxia instance you do not need a wildcard (\*) certificate and thus, in place of DNS01 you can configure [the HTTP01 Ingress solver](https://cert-manager.io/docs/configuration/acme/http01/#configuring-the-http01-ingress-solver) which is much easier to configure. You can follow [this tutorial](https://www.youtube.com/watch?v=hoLUigg4V18).
{% endhint %}

Now we want to create a Kubernetes secret containing our newly obtained certificate: &#x20;

```bash
kubectl create namespace ingress-nginx
DOMAIN=my-domain.net
sudo kubectl create secret tls sill-tls \
    -n ingress-nginx \
    --key /etc/letsencrypt/live/sill.$DOMAIN/privkey.pem \
    --cert /etc/letsencrypt/live/sill.$DOMAIN/fullchain.pem
```

### Ingress controller

We'll install [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) in our cluster ~~but any other ingress controller will do~~.

```bash
cat << EOF > ./ingress-nginx-values.yaml
controller:
  extraArgs:
    default-ssl-certificate: "ingress-nginx/sill-tls"
EOF

helm install ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx \
    --namespace ingress-nginx \
    -f ./ingress-nginx-values.yaml
```
{% endtab %}
{% endtabs %}

At this point we assume that:&#x20;

* You have a Kubernetes cluster and `kubectl` configured
* **sill.my-domain.net** and **\*.sill-tmp.my-domain.net** are pointing to your cluster's external address. **my-domain.net** being a domain that you own. You can customise "**sill**" and "**sill-tmp**" to your likeing, for example you could chose **my-catalog.my-domain.net** and **\*.test-my-catalog.my-domain.net**.
* You have an ingress controller configured with a default TLS certificate for **\*.sill-tmp.my-domain.net** and **sill.my-domain.net**.&#x20;

{% hint style="success" %}
Through out this guide we make as if everything was instantaneous. In reality if you are testing on a small cluster you will need to wait several minutes after hitting `helm install` for the services to be ready. &#x20;

Use `kubectl get pods` to see if your pods are up and ready.&#x20;

![](<.gitbook/assets/image (2).png>)
{% endhint %}

<details>

<summary>(Optional) Make sure that your cluster is ready for the SILL</summary>

To make sure that your Kubernetes cluster is correctly configured let's deploy a test web app on it before deploying the SILL.  &#x20;

<img src=".gitbook/assets/image (19).png" alt="The hello world SPA deployed" data-size="original">

```bash
DOMAIN=my-domain.net

cat << EOF > ./test-spa-values.yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: test-spa.lab.$DOMAIN
image:
  version: "0.4.3"
EOF

helm repo add etalab https://etalab.github.io/helm-charts
helm install test-spa etalab/keycloakify-demo-app -f test-spa-values.yaml
echo "Navigate to https://test-spa.lab.$DOMAIN, see the Hello World"
helm uninstall test-spa
```

</details>

### Installing Keycloak

Let's setup Keycloak to enable users to create account and login to our SILL. &#x20;

For deploying our Keycloak we use [codecentric's helm chart](https://github.com/codecentric/helm-charts/tree/master/charts/keycloak). &#x20;

```bash
helm repo add codecentric https://codecentric.github.io/helm-charts

DOMAIN=my-domain.net
POSTGRESQL_PASSWORD=xxxxx #Replace by a strong password, you will never need it.
# Credentials for logging to https://auth.lab.$DOMAIN/auth
KEYCLOAK_PASSWORD=yyyyyy 

cat << EOF > ./keycloak-values.yaml
image:
  # We use the legacy variant of the image until codecentric update it's helm chart
  tag: "18.0.2-legacy"
replicas: 1
extraInitContainers: |
  - name: realm-ext-provider
    image: curlimages/curl
    imagePullPolicy: IfNotPresent
    command:
      - sh
    args:
      - -c
      - |
        # Extention for AgentConnect (optional)
        curl -L -f -S -o /extensions/keycloak-franceconnect-4.1.0.jar https://github.com/InseeFr/Keycloak-FranceConnect/releases/download/4.1.0/keycloak-franceconnect-4.1.0.jar
        # There is a custom theme published alongside every onyxia-web release
        # The version of the Keycloak theme and the version of onyxia-web don't need 
        # to match but you should update the theme from time to time.  
        # https://github.com/etalab/sill-web/releases
        curl -L -f -S -o /extensions/sill-web.jar https://github.com/etalab/sill-web/releases/download/v0.25.28/standalone-keycloak-theme.jar
    volumeMounts:
      - name: extensions
        mountPath: /extensions
extraVolumeMounts: |
  - name: extensions
    mountPath: /opt/jboss/keycloak/standalone/deployments
extraVolumes: |
  - name: extensions
    emptyDir: {}
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
      -Dkeycloak.profile=preview -XX:+UseContainerSupport -XX:MaxRAMPercentage=50.0 -Djava.net.preferIPv4Stack=true -Djava.awt.headless=true 
ingress:
  enabled: true
  servicePort: http
  annotations:
    kubernetes.io/ingress.class: nginx
    ## Resolve HTTP 502 error using ingress-nginx:
    ## See https://www.ibm.com/support/pages/502-error-ingress-keycloak-response
    nginx.ingress.kubernetes.io/proxy-buffer-size: 128k
  rules:
    - host: "auth.lab.$DOMAIN"
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - auth.lab.$DOMAIN
postgresql:
  postgresqlPassword: $POSTGRESQL_PASSWORD
EOF

helm install keycloak codecentric/keycloak -f keycloak-values.yaml
```

You can now login to the **administration console** of **https://sill-auth.my-domain.net** and login using the credentials you have defined with `KEYCLOAK_USER` and `KEYCLOAK_PASSWORD`.

1. Create a realm called "datalab" (or something else), go to **Realm settings**
   1. On the tab General
      1. _User Profile Enabled_: **On**
   2. On the tab **login**
      1. _User registration_: **On**
      2. _Forgot password_: **On**
      3. _Remember me_: **On**
   3. On the tab **email,** we give an example with **** [AWS SES](https://aws.amazon.com/ses/), if you don't have a SMTP server at hand you can skip this by going to **Authentication** (on the left panel) -> Tab **Required Actions** -> Uncheck "set as default action" **Verify Email**. Be aware that with email verification disable, anyone will be able to sign up to your service.
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
   5. On the tab **Localization**
      1. _Internationalization_: **Enabled**
      2. _Supported locales_: \<Select the languages you wish to support>
2. Create a client called "sill"
   1. _Root URL_: **https://sill-auth.my-domain.net/**
   2. _Valid redirect URIs_: **https://sill.my-domain.net/\***
   3. _Web origins_: **\***
3. In **Authentication** (on the left panel) -> Tab **Required Actions** enable and set as default action **Therms and Conditions.**

Now you want to define a list of email domain allowed to register to your service. &#x20;

Go to **Realm Settings** (on the left panel) -> Tab **User Profile** (this tab shows up only if User Profile is enabled in the General tab and you can enable user profile only if you have started Keycloak with `-Dkeycloak.profile=preview)` -> **JSON Editor**.

Now you can edit the file as suggested in the following DIFF snippet. Be mindful that in this example we only allow emails @gmail.com and @hotmail.com to register you want to edit that. &#x20;

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
 }</code></pre>

Now our Keycloak server is configured.

#### Enabeling AgentConnect

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

To enable agent connect you need to use [this extention](https://github.com/InseeFr/Keycloak-FranceConnect#agent-connect) (I's already loaded in your Keycloak if you look carefully in your `keycloak-values.yaml` file. )

Follow the instructions in the readme of [InseeFr/Keycloak-FranceConnect](https://github.com/InseeFr/Keycloak-FranceConnect#agent-connect). &#x20;

Theses are the information that you'll need to give to the France Connect team to receive your credentials:&#x20;

```
Nom du fournisseur de service: "Socle interministériel de logiciels libres" ( ou SILL pour les intimes )
URL de redirection de connexion: https://sill-auth.my-domain.net/auth/realms/etalab/broker/agentconnect/endpoint
URL de redirection de déconnexion: https://sill-auth.my-domain.net/auth/realms/etalab/broker/agentconnect/endpoint/logout_response
URL du bouton affichant AgentConnect: 
Ici je ne suis pas sûr de comprendre ce que vous me demandez.
Le bouton AgentConnect sera affiché sur la page le login, on accède a la page de login en cliquant sur "Connexion" en haut à droite de l'écran sur https://sill.etalab.gouv.fr/
Scopes désirés: given_name usual_name email organizational_unit
Algorithme de la signature des userinfos: ES256
Algorithme de la signature de l'id token: ES256
```

You'll also need to create a mapper for organizational\_unit -> agencyName. &#x20;

Go to Identity Providers -> Agent Connect -> Mappers -> create

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

### Instantiating the web app

&#x20;helm repo add etalab https://etalab.github.io/helm-charts

```bash

DOMAIN=my-domain.net
SSH_PRIVATE_KEY_NAME=id_ed25521 # ( For example, generated earlyer )
SSH_PRIVATE_KEY="-----BEGIN OPENSSH PRIVATE KEY-----\nxxxx\nxxxx\nxxxx\nAxxxx\nxxxx\n-----END OPENSSH PRIVATE KEY-----\n"

cat << EOF > ./sill-values.yaml
sill:
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
    hosts:
      - host: sill.$DOMAIN
    tls:
      - hosts:
          - sill.$DOMAIN
        secretName: sill-tls
  ui:
    replicaCount: 2
    image:
      # Update on your own term, see releases here: 
      # https://github.com/etalab/sill-web/releases
      # When you update this version number you must
      # also update the api.image.version bellow.
      # See in the following video how to find that the
      # if you are using ui.image.version: 0.25.18
      # you should use api.image.version: 0.22.2
      # https://user-images.githubusercontent.com/6702424/181577434-dbb2c381-4fcf-4c50-b44e-a250339f1f70.mov
      version: 0.25.17
    nodeSelector:
      infra: "true"
    tolerations:
      - key: "infra"
        operator: "Exists"
    env:
      CONFIGURATION: |
         { 
            "headerLinks": [
               { 
                  "label": "code.gouv.fr", 
                  "iconId": "assuredWorkload", 
                  "url": "https://code.gouv.fr" 
               },
               { 
                  "label": { "fr": "Embarquement immédiat", "en": "Immediate boarding" }, 
                  "iconId": "playCircleFilledWhite", 
                  "url": "https://sill-demo.etalab.gouv.fr/" 
               }
            ]
         }
    
  api:
    replicaCount: 1
    image:
      # See comment of ui.image.version to see how to update this field
      version: 0.22.2
    nodeSelector:
      infra: "true"
    tolerations:
      - key: "infra"
        operator: "Exists"
    env:
      CONFIGURATION: |
        {
          "keycloakParams": {
            "url": "https://sill-auth.etalab.gouv.fr/auth",
            "realm": "etalab",
            "clientId": "sill",
            "termsOfServices": "https://sill.etalab.gouv.fr/tos_fr.md",
            "adminPassword": "$KEYCLOAK_PASSWORD" 
          },
          "jwtClaims": {
            "id": "sub",
            "email": "email",
            "agencyName": "agency_name",
            "locale": "locale"
          },
          "dataRepoSshUrl": "git@github.com:etalab/sill-data.git",
          "sshPrivateKeyForGitName": "$SSH_PRIVATE_KEY_NAME",
          "sshPrivateKeyForGit": "$SSH_PRIVATE_KEY" 
        }
EOF

helm install onyxia inseefrlab/onyxia -f etalab-values.yaml
```

&#x20;You can now access `https://sill.my-domain.net`. Congratulations! 🥳

###

