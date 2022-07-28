# 🏁 Installing

This is a step by step guide that will assist you in installing your own instance of [sill.etalab.gouv.fr](https://sill.etalab.gouv.fr). &#x20;

### Provision a Kubernetes cluster

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

### Installing the SILL using helm

In this section we assume that:&#x20;

* You have a Kubernetes cluster and `kubectl` configured
* **sill.my-domain.net** and **\*.sill-tmp.my-domain.net** are pointing to your cluster's external address. **my-domain.net** being a domain that you own. You can customise "**sill**" and "**sill-tmp**" to your liking, for example you could chose **my-catalog.my-domain.net** and **\*.test-my-catalog.my-domain.net**.
* You have an ingress controller configured with a default TLS certificate for **\*.sill-tmp.my-domain.net** and **sill.my-domain.net**. &#x20;

{% hint style="success" %}
Through out this guide we make as if everything was instantaneous. In reality if you are testing on a small cluster you will need to wait several minutes after hitting `helm install` for the services to be ready. &#x20;

Use `kubectl get pods` to see if your pods are up and ready.&#x20;

![](<.gitbook/assets/image (2).png>)
{% endhint %}

<details>

<summary>(Optional) Make sure that your cluster is ready for Onyxia</summary>

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

```bash
helm repo add etalab https://etalab.github.io/helm-charts

DOMAIN=my-domain.net
SSH_PRIVATE_KEY_NAME=id_ed25521 # ( For example )
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

### Enabling user authentication

At the moment there is no authentication process, everyone can access our platform and and start services. &#x20;

Let's setup Keycloak to enable users to create account and login to our Onyxia. &#x20;

<details>

<summary>Notes if you already have a Keycloak server</summary>

If you already have a Keycloak server it is up to you to pick from this guide what is rellevent to you. &#x20;

Main takeway is that you probably want to load the Onyxia custom Keycloak theme and enable `-Dkeycloak.profile=preview` in order to be able to enforce that usernames are well formatted and define an accept list of email domains allowed to create account on your Onyxia instance. &#x20;

You probably want to enable [Terms and Conditions as required actions](https://docs.keycloakify.dev/terms-and-conditions).

That out of the way, note that you can configure onyxia-web to integrate with your existing Keycloak server, you just need to set some dedicated environment variable in the `values.yaml` of the onyxia helm chart. Example: &#x20;

```yaml
 ui:
   image:
     version: "0.56.5"
  env:
    # Available env are documented here: https://github.com/InseeFrLab/onyxia-web/blob/main/.env
    KEYCLOAK_URL: https://auth.lab.my-domain.net/auth
    KEYCLOAK_CLIENT_ID: onyxia
    KEYCLOAK_REALM: datalab
    JWT_EMAIL_CLAIM: email
    JWT_FAMILY_NAME_CLAIM: family_name
    JWT_FIRST_NAME_CLAIM: given_name
    JWT_USERNAME_CLAIM: preferred_username
    JWT_LOCALE_CLAIM: locale
```

</details>

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
        # There is a custom theme published alongside every onyxia-web release
        # The version of the Keycloak theme and the version of onyxia-web don't need 
        # to match but you should update the theme from time to time.  
        # https://github.com/InseeFrLab/onyxia-web/releases
        curl -L -f -S -o /extensions/onyxia-web.jar https://github.com/InseeFrLab/onyxia-web/releases/download/v0.56.6/standalone-keycloak-theme.jar
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

You can now login to the **administration console** of **https://auth.lab.my-domain.net** and login using the credentials you have defined with `KEYCLOAK_USER` and `KEYCLOAK_PASSWORD`.

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
      1. _Login theme_: **onyxia-web** (you can also select the login theme on a per client basis)
      2. _Email theme_: **onyxia-web**
   5. On the tab **Localization**
      1. _Internationalization_: **Enabled**
      2. _Supported locales_: \<Select the languages you wish to support>
2. Create a client called "onyxia"
   1. _Root URL_: **https://onyxia.my-domain.net/**
   2. _Valid redirect URIs_: **https://onyxia.my-domain.net/\***
   3. _Web origins_: **\***
   4. Login theme: **onyxia-web**
3. In **Authentication** (on the left panel) -> Tab **Required Actions** enable and set as default action **Therms and Conditions.**

Now you want to ensure that the username chosen by your users complies with Onyxia requirement (only alphanumerical characters) and define a list of email domain allowed to register to your service. &#x20;

Go to **Realm Settings** (on the left panel) -> Tab **User Profile** (this tab shows up only if User Profile is enabled in the General tab and you can enable user profile only if you have started Keycloak with `-Dkeycloak.profile=preview)` -> **JSON Editor**.

Now you can edit the file as suggested in the following DIFF snippet. Be mindful that in this example we only allow emails @gmail.com and @hotmail.com to register you want to edit that. &#x20;

```diff
{
  "attributes": [
    {
      "name": "username",
      "displayName": "${username}",
      "validations": {
        "length": {
          "min": 3,
          "max": 255
        },
+       "pattern": {
+         "error-message": "${alphanumericalCharsOnly}",
+         "pattern": "^[a-zA-Z0-9]*$"
+       },
        "username-prohibited-characters": {}
      }
    },
    {
      "name": "email",
      "displayName": "${email}",
      "validations": {
        "email": {},
+       "pattern": {
+         "pattern": "^[^@]+@([^.]+\\.)*((gmail\\.com)|(hotmail\\.com))$"
+       },
        "length": {
          "max": 255
        }
      }
    },
    {
      "name": "firstName",
      "displayName": "${firstName}",
      "required": {
        "roles": [
          "user"
        ]
      },
      "permissions": {
        "view": [
          "admin",
          "user"
        ],
        "edit": [
          "admin",
          "user"
        ]
      },
      "validations": {
        "length": {
          "max": 255
        },
        "person-name-prohibited-characters": {}
      }
    },
    {
      "name": "lastName",
      "displayName": "${lastName}",
      "required": {
        "roles": [
          "user"
        ]
      },
      "permissions": {
        "view": [
          "admin",
          "user"
        ],
        "edit": [
          "admin",
          "user"
        ]
      },
      "validations": {
        "length": {
          "max": 255
        },
        "person-name-prohibited-characters": {}
      }
    }
  ]
}
```

Now our Keycloak server is fully configured we just need to update our Onyxia deployment to let it know about it. &#x20;

Update the `onyxia-values.yaml` file  that you created previously, don't forget to replace all the occurence of **my-domain.net** by your actual domain. &#x20;

Don't forget as well to remplace the terms of services of the [sspcloud](https://www.sspcloud.fr) by your own terms of services. CORS should be enabled on those `.md` links (`Access-Control-Allow-Origin: *`).

```diff
+serviceAccount:
+  clusterAdmin: true
 ingress:
   enabled: true
   annotations:
     kubernetes.io/ingress.class: nginx
   hosts:
     - host: onyxia.my-domain.net
 ui:
   image:
     # Update on your own therm but update!
     # https://hub.docker.com/r/inseefrlab/onyxia-api/tags
     version: 0.56.6
+  env:
+    KEYCLOAK_REALM: datalab
+    KEYCLOAK_URL: https://auth.lab.my-domain.net/auth
+    TERMS_OF_SERVICES: |
+      { "en": "https://www.sspcloud.fr/tos_en.md", "fr": "https://www.sspcloud.fr/tos_fr.md" }
 api:
   image:
     # Same here
     # https://hub.docker.com/r/inseefrlab/onyxia-api/tags
     version: multi-arch
   env:
     security.cors.allowed_origins: "http://localhost:3000"
+    authentication.mode: openidconnect
+    keycloak.realm: datalab
+    keycloak.auth-server-url: https://auth.lab.my-domain.net/auth
   regions:
     [
        {
           "id":"demo",
           "name":"Demo",
           "description":"This is a demo region, feel free to try Onyxia !",
           "services":{
              "type":"KUBERNETES",
-             "singleNamespace": true,
+             "singleNamespace": false,
              "namespacePrefix":"user-",
              "usernamePrefix":"oidc-",
              "groupNamespacePrefix":"projet-",
              "groupPrefix":"oidc-",
              "authenticationMode":"admin",
              "expose":{
                 "domain":"lab.my-domain.net"
              },
              "monitoring":{
                 "URLPattern":"todo"
              },
              "cloudshell":{
                 "catalogId":"inseefrlab-helm-charts-datascience",
                 "packageName":"cloudshell"
              },
              "initScript":"https://inseefrlab.github.io/onyxia/onyxia-init.sh"
           },
           "data":{
              "S3":{
                 "URL":"todo",
                 "monitoring":{
                    "URLPattern":"todo"
                 }
              }
           },
           "auth":{
              "type":"openidconnect"
           },
           "location":{
              "lat":48.8164,
              "long":2.3174,
              "name":"Montrouge (France)"
           }
        }
     ]
```

Now that you have updated `onyxia-values.yaml` restart onyxia-web with the new configuration.

```bash
helm upgrade onyxia inseefrlab/onyxia -f onyxia-values.yaml
```

Now your users should be able to create account, log-in, and start services on their own Kubernetes namespace.

### Minio, Vault

TODO; [Refer to the legacy documentation.](https://github.com/InseeFrLab/onyxia/tree/main/step-by-step#set-up-authentication-openidconnect)
