# 📐 Architecture

## Main rules

* [`src/ui`](https://github.com/InseeFrLab/onyxia-web/tree/main/src/ui) contains the React application, it's the UI of the app.
  * All the import of src/core should be made in [`src/ui/coreApi`](https://github.com/InseeFrLab/onyxia-web/tree/main/src/ui/coreApi).
* [`src/core`](https://github.com/InseeFrLab/onyxia-web/tree/main/src/core) contains the 🧠  of the app.
  * Nothing in the `src/core` directory should relate to React. A concept like react hooks for example is out of scope for the src/core directory.
  * `src/core` should never import anything from `src/ui`, even types.
  * It should be possible for example to port onyxia-web to Vue.js or React Native without changing anything to the `src/core` directory.
  * The goal of `src/core` is to expose an API that serves the UI.
  * The API exposed should be reactive. We should not expose to the UI functions that returns promises, instead, the functions we expose should update states and the UI should react to these states updates.

{% hint style="warning" %}
The `src/js` directory is legacy. It will be removed soon.
{% endhint %}

## Clean Archi

* Whenever we need to interact with the infrastructure we define a port in [`src/core/port`](https://github.com/InseeFrLab/onyxia-web/tree/main/src/core/ports). A port is only a type definition. In our case the infrastructure is: the Keycloak server, the Vault server, the Minio server and a Kubernetes API (Onyxia-API).
* In [`src/core/secondaryAdapter`](https://github.com/InseeFrLab/onyxia-web/tree/main/src/core/secondaryAdapters) are the implementations of the ports. For each port we should have at least two implementations, a dummy and a real one. It enabled the app to still run, be it in degraded mode, if one piece of the infrastructure is missing. Say we don’t have a Vault server we should still be able to launch containers.
* In [`src/lib/usecases`](https://github.com/InseeFrLab/onyxia-web/tree/main/src/core/usecases) we expose APIs for the UI to consume.

## In practice

Let's say we want to create a new page in onyxia-web where users can type in a repo name and get the current number of stars the repo has on GitHub.

{% hint style="info" %}
UPDATE: This video remain relevant but please not that the clean archi setup have been considerably improved in latest releases. [A dedicated repo](https://github.com/garronej/clean-redux) have been created to explain it in detail.&#x20;

Main take-way is that `app` have been renamed `ui` and `lib` have been renamed `core`.
{% endhint %}

{% embed url="https://youtu.be/RDxAag3Iq0o" %}

{% hint style="info" %}
You might wonder why some values, instead of being redux state, are returned by thunks functions. &#x20;

For example, it might seem more natural to do: &#x20;

```tsx
const { isUserLoggedIn } = useSelector(state => state.userAuthentication);
```

Instead of what we actually do, which is: &#x20;

```tsx
const { userAuthenticationThunks } = useThunks();
const isUserLoggedIn = userAuthenticationThunks.getIsUserLoggedIn();
```

However the rule is to never store as a redux state, values that are not susceptible to change. Redux states are values that we observe, any redux state changes should trigger a re-render of the React components that uses them. Conversely, there is no need to observe a value that will never change. We can get it once and never again, get it in a callback or wherever.&#x20;

But, you may object, users do login and logout, `isUserLoggedIn` is not a constant!&#x20;

Actually, from the standpoint of the web app, it is. When a user that isn't authenticated click on the login button, it is being redirected away. When he returns to the app everything is reloaded from scratch.
{% endhint %}

Now let's say we want the search to be restricted to a given GitHub organization. (Example: InseeFrLab.) The GitHub organization should be specified as an environment variable by the person in charge of deploying Onyxia. e.g.:

```yaml
  UI:
    image:
      name: inseefrlab/onyxia-web
      version: 0.15.13
    env:
      MINIO_URL: https://minio.lab.sspcloud.fr
      VAULT_URL: https://vault.lab.sspcloud.fr
      OIDC_URL: https://auth.lab.sspcloud.fr/auth
      OIDC_REALM: sspcloud
      TITLE: SSP Cloud
      ORG_NAME: InseeFrLab #<==========
      
```

If no `ORG_NAME` is provided by the administrator, the app should always show 999 stars for any repo name queried.

{% embed url="https://youtu.be/eaU-tYFzWwA" %}

## Another example: Recording user's GitLab token

Currently users can save their GitHub Personal access token in their Onyxia account but not yet their GitLab token. Let's see how we would implement that.

{% embed url="https://www.youtube.com/watch?v=WVFKCR1QfVk" %}

## How to deal with project switching

The easy action to take when the user selects another project is to simply reload the page (`windows.location.reload()`). We want to avoid doing this to enable what we call "_hot projet swiping_":&#x20;

![The page is not reloaded when changing the project](https://user-images.githubusercontent.com/6702424/147413744-480235af-53cc-4b4d-a69a-7e9e73a79407.gif)

To implement this behavior you have to leverage the evtAction middleware from clean-redux. It enabled to register functions to be run when certain actions are dispatched.

{% hint style="info" %}
Unlike the other video, the following one is voiced. Find the relevant code [here](https://github.com/InseeFrLab/onyxia-web/blob/61b4d660faebefacc9e963c506b707c04d57521f/src/core/usecases/runningService.ts#L316-L332).
{% endhint %}

{% embed url="https://youtu.be/TWDHBxceH0Q" %}
