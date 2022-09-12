---
description: Contributing to the helm charts
---

# 🔬 Help maintain the catalog of service



Every Onyxia instance may or may not have it's own catalog. &#x20;

If you take for example [this instance](https://datalab.sspcloud.fr/catalog) it offers two catalogs, [InseeFrLab datascience](https://github.com/InseeFrLab/helm-charts-datascience) and [InseeFrLab R Shiny apps](https://github.com/InseeFrLab/helm-charts-shiny-apps). &#x20;

You can always find the source of the catalog by clicking on the "contribute to the... " link.

![https://datalab.sspcloud.fr/catalog](<../.gitbook/assets/image (7).png>)

If you take [this other instance](https://sill-demo.etalab.gouv.fr), it has only one catalog, [helm-charts-sill](https://github.com/etalab/helm-charts-sill).

![https://sill-demo.etalab.gouv.fr/catalog](<../.gitbook/assets/image (20).png>)

The available catalogs in a given Onyxia instance are configured at install time, example with datalab.sspcloud.fr: &#x20;

```bash
helm repo add inseefrlab https://inseefrlab.github.io/helm-charts

DOMAIN=my-domain.net

cat << EOF > ./onyxia-values.yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: onyxia.$DOMAIN
ui:
  image:
    version: 0.56.6
api:
  image:
    version: v0.12
  catalogs: 
    [
      {
        "id": "inseefrlab-helm-charts-datascience",
        "name": "Inseefrlab datascience",
        "description": "Services for datascientists.",
        "maintainer": "innovation@insee.fr",
        "location": "https://inseefrlab.github.io/helm-charts-datascience",
        "status": "PROD",
        "type": "helm"
      },
      {
        "id": "inseefrlab-helm-charts-shiny-apps",
        "name": "Inseefrlab R Shiny apps",
        "description": "R Shiny apps as services.",
        "maintainer": "innovation@insee.fr",
        "location": "https://inseefrlab.github.io/helm-charts-shiny-apps",
        "status": "PROD",
        "type": "helm"
      },
    ]
  regions: 
    [
      {
        "services":{
          "expose":{
            "domain":"lab.$DOMAIN"
          }
        }
      }
    ]
EOF

helm install onyxia inseefrlab/onyxia -f onyxia-values.yaml
```

In order to contribute you have to be familiar with [Helm](https://helm.sh/) and to be familiar with Helm you need to be familiar with [Kubernetes objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/). &#x20;

In Onyxia we use the `values.shema.json` be  to know what options should be displayed to the user at [the service configuration step](https://user-images.githubusercontent.com/6702424/177571819-f2e1b4ef-ecd1-479b-a5a1-658d87d7c7c0.png) and what default value Onyxia should inject. &#x20;

![https://helm.sh/docs/topics/charts/#the-chart-file-structure](<../.gitbook/assets/image (8) (1).png>)

Let's consider a sample of the `values.scema.json` of the InseeFrLab/helm-charts-datascience's Jupyter chart: &#x20;

```javascript
"git": {
    "description": "Git user configuration",
    "type": "object",
    "properties": {
        "enabled": {
            "type": "boolean",
            "description": "Add git config inside your environment",
            "default": true
        },
        "name": {
            "type": "string",
            "description": "user name for git",
            "default": "",
            "x-onyxia": {
                "overwriteDefaultWith": "git.name"
            },
            "hidden": {
                "value": false,
                "path": "git/enabled"
            }
        },
        "email": {
            "type": "string",
            "description": "user email for git",
            "default": "",
            "x-onyxia": {
                "overwriteDefaultWith": "git.email"
            },
            "hidden": {
                "value": false,
                "path": "git/enabled"
            }
        },
        "cache": {
            "type": "string",
            "description": "duration in seconds of the credentials cache duration",
            "default": "",
            "x-onyxia": {
                "overwriteDefaultWith": "git.credentials_cache_duration"
            },
            "hidden": {
                "value": false,
                "path": "git/enabled"
            }
        },
        "token": {
            "type": "string",
            "description": "personal access token",
            "default": "",
            "x-onyxia": {
                "overwriteDefaultWith": "git.token"
            },
            "hidden": {
                "value": false,
                "path": "git/enabled"
            }
        },
        "repository": {
            "type": "string",
            "description": "Repository url",
            "default": "",
            "hidden": {
                "value": false,
                "path": "git/enabled"
            }
        },
        "branch": {
            "type": "string",
            "description": "Brach automatically checkout",
            "default": "",
            "hidden": {
                "value": "",
                "path": "git/repository"
            }
        }
    }
},
```

And it translate into this:

{% embed url="https://user-images.githubusercontent.com/6702424/177571819-f2e1b4ef-ecd1-479b-a5a1-658d87d7c7c0.png" %}

Note the `"git.name"`, `"git.email"` and `"git.token"`, this enables [onyxia-web](https://github.com/InseeFrLab/onyxia-web) to pre fill the fields. &#x20;

If the user took the time to fill it's profile information, [onyxia-web](https://github.com/InseeFrLab/onyxia-web) know what is the Git **username**, **email** and **personal access token** of the user. &#x20;

![The onyxia user profile](<../.gitbook/assets/image (14).png>)

[Here are the values](https://github.com/InseeFrLab/onyxia-web/blob/ea2580954d50b1acedc03867353b6c26ab27eb79/src/core/ports/OnyxiaApiClient.ts#L139-L190) that you can use in the `overwriteDefaultWith` field :&#x20;

```typescript
export type OnyxiaValues = {
    user: {
        idep: string;
        name: string;
        email: string;
        password: string;
        ip: string;
    };
    project: {
        id: string;
        password: string;
        basic: string;
    };
    git: {
        name: string;
        email: string;
        credentials_cache_duration: number;
        token?: string;
    };
    vault: {
        VAULT_ADDR: string;
        VAULT_TOKEN: string;
        VAULT_MOUNT: string;
        VAULT_TOP_DIR: string;
    };
    kaggleApiToken?: string;
    s3: {
        AWS_ACCESS_KEY_ID: string;
        AWS_SECRET_ACCESS_KEY: string;
        AWS_SESSION_TOKEN: string;
        AWS_DEFAULT_REGION: string;
        AWS_S3_ENDPOINT: string;
        AWS_BUCKET_NAME: string;
        port: number;
    };
    region: {
        defaultIpProtection?: boolean;
        defaultNetworkPolicy?: boolean;
        allowedURIPattern: string;
        kafka?: {
          url: string;
          topicName: string;
        };
    };
    k8s: {
        domain: string;
        randomSubdomain: string;
        initScriptUrl: string;
    };
};
```

You can also concatenate string values using [mustache](https://mustache.github.io/) syntax.

```javascript
"hostname": {
  "type": "string",
  "form": true,
  "title": "Hostname",
  "x-onyxia": {
    "hidden": true,
    "overwriteDefaultWith": "{{project.id}}-{{k8s.randomSubdomain}}.{{k8s.domain}}"
  }
}
```

You now have all the relevent information to submit PR on the exsisting catalogs or even to create your own. &#x20;

Remember that a helm chart repository is nothing more than a GitHub repo with a special [github Action setup](https://github.com/marketplace/actions/helm-deploy) to publish the charts on GitHub Pages. &#x20;

If you are looking for a repo to start from have a look at [this one](https://github.com/etalab/helm-charts-sill), it has a directory where you can put the icons of your services. &#x20;
