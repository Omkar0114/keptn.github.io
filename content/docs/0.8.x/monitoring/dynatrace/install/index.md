---
title: Install
description: Setup Dynatrace monitoring and Keptn integration
weight: 1
icon: setup
---

## Deploy Dynatrace Operator on Kubernetes

Bring your Dynatrace SaaS or Dynatrace-managed tenant. If you do not have a Dynatrace tenant, sign up for a [free trial](https://www.dynatrace.com/trial/) or a [developer account](https://www.dynatrace.com/developer/).

To setup Dynatrace monitoring on your Kubernetes cluster, please follow the official Dynatrace documentation on [Deploy Dynatrace Operator](https://www.dynatrace.com/support/help/technology-support/container-platforms/kubernetes/monitor-kubernetes-environments/).

## Install Dynatrace Keptn integration

### 1. Gather Dynatrace and Keptn Credentials

To function correctly, the *dynatrace-service* requires access to a Dynatrace tenant and to the Keptn API.

*  The credentials for the Dynatrace tenant include `DT_API_TOKEN` and `DT_TENANT`:

    * The `DT_TENANT` has to be set according to the appropriate pattern:
        - Dynatrace SaaS tenant: `{your-environment-id}.live.dynatrace.com`
        - Dynatrace-managed tenant: `{your-domain}/e/{your-environment-id}`

    * To create a Dynatrace API token `DT_API_TOKEN`, log in to your Dynatrace tenant and go to **Settings > Integration > Dynatrace API**. In this settings page, create a new API token with the following permissions:
        - Access problem and event feed, metrics, and topology
        - Read log content
        - Read configuration
        - Write configuration
        - Capture request data
        - Read metrics
        - Ingest metrics
        - Read entities

        {{< popup_image
        link="./assets/dt_api_token.png"
        caption="Dynatrace API Token"
        width="500px">}}

* The credentials for access to Keptn include `KEPTN_API_URL`, `KEPTN_API_TOKEN` and optionally `KEPTN_BRIDGE_URL`:

    * To get the values for `KEPTN_API_URL` (aka. `KEPTN_ENDPOINT`) and `KEPTN_API_TOKEN`, please see [Authenticate Keptn CLI](../../../operate/install/#authenticate-keptn-cli).

    * If you would like to use backlinks from your Dynatrace tenant to the Keptn Bridge, provide the service with `KEPTN_BRIDGE_URL`. For further details about this value, please see [Authenticate Keptn Bridge](../../../operate/install/#authenticate-keptn-bridge).

* If running on a Unix/Linux based system, you can use environment variables to set the values of the credentials. It is also fine to just replace the variables with values in the `kubectl` and `helm` commands in the following sections.

    ```console
    DT_API_TOKEN=<DT_API_TOKEN>
    DT_TENANT=<DT_TENANT>
    KEPTN_API_URL=<KEPTN_API_URL>
    KEPTN_API_TOKEN=<KEPTN_API_TOKEN>
    KEPTN_BRIDGE_URL=<KEPTN_BRIDGE_URL> # optional
    ```

### 2. Create a secret with credentials

* Create a secret (named `dynatrace` by default) containing the credentials for the Dynatrace Tenant (`DT_API_TOKEN` and `DT_TENANT`) and optionally for the Keptn API (`KEPTN_API_URL`, `KEPTN_API_TOKEN` and `KEPTN_BRIDGE_URL`).

    ```console
    kubectl -n keptn create secret generic dynatrace \
    --from-literal="DT_API_TOKEN=$DT_API_TOKEN" \
    --from-literal="DT_TENANT=$DT_TENANT" \
    --from-literal="KEPTN_API_URL=$KEPTN_API_URL" \
    --from-literal="KEPTN_API_TOKEN=$KEPTN_API_TOKEN" \
    --from-literal="KEPTN_BRIDGE_URL=$KEPTN_BRIDGE_URL" \
    -oyaml --dry-run=client | kubectl replace -f -
    ```

* If the Keptn credentials are omitted from this secret, they must be set
when applying the Helm chart (see below).

### 3. Deploy the Dynatrace Keptn integration

The Dynatrace integration into Keptn is handled by the *dynatrace-service*.

* Specify the version of the dynatrace-service you want to deploy. Please see the [compatibility matrix](https://github.com/keptn-contrib/dynatrace-service#compatibility-matrix) of the dynatrace-service to pick the version that works with your Keptn.

    ```console
    VERSION=<VERSION>   # e.g.: VERSION=0.12.0
    ```

*  To install the *dynatrace-service*, execute:

    ```console
    helm upgrade --install dynatrace-service -n keptn https://github.com/keptn-contrib/dynatrace-service/releases/download/$VERSION/dynatrace-service-$VERSION.tgz
    ```

* This installs the `dynatrace-service` in the `keptn` namespace, which you can verify using:

    ```console
    kubectl -n keptn get deployment dynatrace-service -o wide
    kubectl -n keptn get pods -l run=dynatrace-service
    ```

**Note**: If the `KEPTN_API_URL`, `KEPTN_API_TOKEN` and optionally `KEPTN_BRIDGE_URL` are not provided via a secret (see above), they must be provided using the variables `dynatraceService.config.keptnApiUrl` and `dynatraceService.config.keptnBridgeUrl`, i.e. by appending `--set dynatraceService.config.keptnApiUrl=$KEPTN_API_URL --set dynatraceService.config.keptnBridgeUrl=$KEPTN_BRIDGE_URL` to the `helm upgrade` command used to install the service. Moreover, the `KEPTN_API_TOKEN` is read from the `keptn-api-token` secret (i.e. the secret from the control-plane).

## Verify Dynatrace configuration

When you execute the [keptn configure monitoring](../../../reference/cli/commands/keptn_configure_monitoring/) command, the *dynatrace-service* will configure the Dynatrace tenant by creating *tagging rules*, a *problem notification*, an *alerting profile* as well as a project-specific *dashboard* and *management zone*.

- *Tagging rules:* When you navigate to **Settings > Tags > Automatically applied tags** in your Dynatrace tenant, you will find following tagging rules:
    - keptn_deployment
    - keptn_project
    - keptn_service
    - keptn_stage

    This means that Dynatrace will automatically apply tags to your onboarded services.

- *Problem notification:* A problem notification has been set up to inform Keptn of any problems with your services to allow auto-remediation. You can check the problem notification by navigating to **Settings > Integration > Problem notifications** and you will find a **keptn remediation** problem notification.

- *Alerting profile:* An alerting profile with all problems set to *0 minutes* (immediate) is created. You can review this profile by navigating to **Settings > Alerting > Alerting profiles**.

- *Dashboard and Management zone:* When creating a new Keptn project or executing the [keptn configure monitoring](../../../reference/cli/commands/keptn_configure_monitoring/) command for a particular project (see Note 1), a dashboard and management zone will be generated reflecting the environment as specified in the shipyard.

## Notes

**Note 1:** If you already have created a project using Keptn and would like to enable Dynatrace monitoring for that project, please execute the following command:

```console
keptn configure monitoring dynatrace --project=PROJECTNAME
```

**Note 2:** To monitor the services that are already onboarded in the **dev**, **staging**, and **production** namespace, make sure to restart the pods. If you defined different environments in your shipyard, please adjust the parameters accordingly.

```console
kubectl delete pods --all --namespace=sockshop-dev
```
```console
kubectl delete pods --all --namespace=sockshop-staging
```
```console
kubectl delete pods --all --namespace=sockshop-production
```

**Note 3:** If the nodes in your cluster run on *Container-Optimized OS (cos)* (default for GKE), the Dynatrace OneAgent might not work properly, and another step is necessary. To verify that the OneAgent does not work properly, the output of `kubectl get pods -n dynatrace` might look as follows:

```console
NAME                                           READY   STATUS             RESTARTS   AGE
dynatrace-oneagent-operator-7f477bf78d-dgwb6   1/1     Running            0          8m21s
oneagent-b22m4                                 0/1     Error              6          8m15s
oneagent-k7jn6                                 0/1     CrashLoopBackOff   6          8m15s
```

1. This means that after the initial setup you need to edit the OneAgent custom resource in the Dynatrace namespace and add the following entry to the env section:

        env:
        - name: ONEAGENT_ENABLE_VOLUME_STORAGE
          value: "true"

1. To edit the OneAgent custom resource:

    ```console
    kubectl edit oneagent -n dynatrace
    ```

1. Finally, don't forget to restart the pods as described in **Note 2** above.
