---
description: >-
  Hub for everything about the application behind the list of recommended free
  software for the French public sector (SILL).
---

# 🎯 Index

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption><p>Screenshot of the mail page of the WebApp</p></figcaption></figure>

### Source code

* The source Markdown for this documentation website. You can also request an access to the [the GitBook](https://app.gitbook.com/o/w6D6SnLwCXQaMMSzcTvp/s/WfLZKgyNVcGm8CUpiWb0/)
* Frontend: [etalab/sill-web](https://github.com/etalab/sill-web)
* Backend: [etalab/sill-api](https://github.com/etalab/sill-api)
* Data (source of truth, **private repository**): [etalab/sill-data](https://github.com/etalab/sill-data)
  * The compiled public data are available at [etalab/sill-data#build](https://github.com/etalab/sill-data/tree/build) (the [`build`](https://github.com/etalab/sill-data/tree/build) branch of the [etalab/sill-data](https://github.com/etalab/sill-data) repo)
  * The data (without the referent information) can be downloaded here: [sill.json](https://sill.etalab.gouv.fr/api/sill.json)
* Helm package of the SILL WebApp: [etalab/helm-charts/charts/sill](https://github.com/etalab/helm-charts/tree/main/charts/sill) (for installing the WebApp on a Kubernetes cluster with `helm install sill`).
* GitOps deployment: [etalab/paris-sspcloud/apps/sill](https://github.com/etalab/paris-sspcloud/tree/main/apps/sill) (It's a private repo that is monitored by ArgoCD)

### Deployment

#### SILL

* Prod: [https://sill.etalab.gouv.fr](https://sill.etalab.gouv.fr/software)
* Dev: [https://sill-dev.lab.sspcloud.fr](https://sill-dev.lab.sspcloud.fr/) (With test data: [etalab/sill-data-test](https://github.com/etalab/sill-data-test))

#### SILL Demos (aka "Embarquement immédiat")

* [https://sill-demo.etalab.gouv.fr](https://sill-demo.etalab.gouv.fr/)
* Helm charts of the testable software of the SILL: [etalab/helm-charts-sill](https://github.com/etalab/helm-charts-sill)
* GitOps deployment: [etalab/paris-sspcloud/apps/onyxia-sill](https://github.com/etalab/paris-sspcloud/tree/main/apps/onyxia-sill) (private repo)
