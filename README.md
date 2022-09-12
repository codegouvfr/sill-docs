# SILL Hub

Hub for everything about the application behind the list of recommended free software for the French public sector ([SILL](https://sill.etalab.gouv.fr)).

- Documentation website: https://etalab-2.gitbook.io/sill/, the source are in the `docs/` directory. [It's a GitBook](https://app.gitbook.com/o/w6D6SnLwCXQaMMSzcTvp/s/WfLZKgyNVcGm8CUpiWb0/)  
- Frontend: [etalab/sill-web](https://github.com/etalab/sill-web)
- Backend: [etalab/sill-api](https://github.com/etalab/sill-api)
- Data (source of truth, **private repository**): [etalab/sill-data](https://github.com/etalab/sill-data)
- Compiled public data: [sill.json](https://sill.etalab.gouv.fr/api/sill.json)
- Helm package of the SILL WebApp: [etalab/helm-charts](https://github.com/etalab/helm-charts/tree/main/charts/sill)
- GitOps deployment: [etalab/paris-sspcloud](https://github.com/etalab/paris-sspcloud/tree/main/apps/sill) (private repo)

## Deployment

### SILL

- Prod: https://sill.etalab.gouv.fr
- Dev: https://sill-dev.lab.sspcloud.fr (With test data: [etalab/sill-data-test](https://github.com/etalab/sill-data-test))

## SILL Demos (aka "Embarquement immédiat")

- https://sill-demo.etalab.gouv.fr
- Helm charts of the testable software of the SILL: [etalab/helm-charts-sill](https://github.com/etalab/helm-charts-sill)
- GitOps deployment: [etalab/paris-sspcloud](https://github.com/etalab/paris-sspcloud/tree/main/apps/onyxia-sill) (private repo)

## License

Direction interministérielle du numérique and contributors of the repository, 2022.

The documentation in this repository is published under [licence Ouverte 2.0](LICENSES/LICENSE.Etalab-2.0.md).
