# Openshift Manifests via Helm Charts

[TOC]

> This branch contains Helm charts and configuration maps for deploying applications on Red Hat OpenShift
Container Platform (RH OCP) on Development (uat) environment.


## Repository layout for development environment (dev) branch
- `{application-name}/` — Each application has its own directory containing Helm charts and configuration maps.
- `templates/` — This directory contains the Helm chart template files for the application eg deployments , service , hpa  and route .
- `configmaps/` — This directory contains the ConfigMaps required for the application deployment , each application has its own directory for config maps. Common config maps are stored in a `common-config` directory.

### To Create Application Artifacts via Helm refer to the following repository:
- http://gitlab.example.com/your-org/your-helm-repo.git


## Steps to Create Application Artifacts via Helm Charts

1. Clone the repository:
   ```bash
   git clone <REPO_URL>
   cd <REPO_NAME>
   ```

2. switch to the `dev` branch:
   ```bash
   git checkout dev
   ```

3. Create a new directory for your application:
   ```bash
   mkdir {application-name}
    cd {application-name}
    ```
4. Create Values.yaml file for your application with the necessary configurations.

5. Create a new directory for configmaps in config maps folder :
   ```bash
   mkdir configmaps/{application-name}
   ```
6. Create ConfigMap yaml files in the newly created directory with the necessary configurations.

7. Commit and push your changes to the repository:
   ```bash
   git add .
   git commit -m "Add Helm chart and ConfigMaps for {application-name}"
   git push origin dev
   ```

## Steps to add the Application in Openshift GitOps ApplicationSet

1. Navigate to the OpenShift console for your cluster.

2. Select "Installed Operators" from the left-hand menu.

3. Select `openshift-gitops` project/namespace.

4. Click on "OpenShift GitOps" under the `Installed Operators` section.

5. Edit the ApplicationSet for the dev environment (use a name that matches your app, e.g. `{application-name}-dev`).

6. Open the ApplicationSet for editing and add an entry for your application in the generators list. Use `{application-name}` placeholders, for example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: {application-name}-dev
  namespace: openshift-gitops
spec:
  generators:
  - list:
      elements:
        - name: {application-name}
template:
  # ... Application template ...
```

7. Save the changes to the ApplicationSet.
