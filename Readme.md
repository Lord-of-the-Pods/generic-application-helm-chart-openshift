# Openshift Manifests via Helm Charts

This repository provides a Helm chart for a generic OpenShift application and a step-by-step workflow to publish the chart as a Helm repository using GitHub Pages and deploy multiple applications using Argo CD ApplicationSet.

Checklist
- [ ] Package the chart and publish it to GitHub Pages (`docs/`)
- [ ] Enable GitHub Pages for this repository (serve `docs/` from `main`)
- [ ] Add the published Helm repo to your local Helm client
- [ ] Create an Argo CD ApplicationSet that deploys multiple apps from multiple sources (Helm repo + Git values)

Overview
This README shows how to:
- Publish the Helm chart in this repo as a Helm repository served from GitHub Pages.
- Add the Helm repo locally and validate the chart.
- Use an Argo CD `ApplicationSet` to deploy multiple applications that share the same chart (or different charts) and different values or sources.

Prerequisites
- Helm v3 installed
- kubectl configured to the cluster you will use
- argocd CLI (optional) or Argo CD GUI access
- GitHub repository `owner/repo` (this repo); permission to push and enable Pages

1) Publish the Helm chart to GitHub Pages (docs/)
1.1 Package the chart (run from the chart directory):

```bash
cd generic-application-helm-chart-openshift
helm package . -d ../docs
```

1.2 Create or update the repository index pointing to the GitHub Pages URL (replace `<owner>` and `<repo>`):

```bash
# from repo root, put the index in docs/ and point to your GitHub Pages URL
helm repo index docs --url https://<owner>.github.io/<repo>
```

1.3 Ensure `docs/` contains:
- `openshift-application-<version>.tgz`
- `index.yaml`
- `.nojekyll` (optional but recommended)

1.4 Commit and push `docs/` and enable GitHub Pages to serve `docs/` from the `main` branch.

2) Add the Helm repo locally and validate
```bash
helm repo add mycharts https://<owner>.github.io/<repo>
helm repo update
helm search repo mycharts/openshift-application --versions
helm pull mycharts/openshift-application --version 0.1.0
```

3) Deploy multiple apps with Argo CD ApplicationSet

Why ApplicationSet?
ApplicationSet lets you generate multiple Argo `Application` resources from a single template and a generator. Using the `list` generator (or other generators like `git`, `matrix`, and `clusters`), you can create a set of Applications that deploy the same Helm chart with different values or deploy different charts from different sources.

3.1 Example: multiple applications from a Helm repo
Below is an example `ApplicationSet` that creates multiple Applications from the hosted Helm repo. Each element in the `list` generator provides per-application parameters (name, target namespace, chart version, etc.). The template uses those parameters to render an Argo `Application` that installs the Helm chart into a specific namespace.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: openshift-apps-set
  namespace: openshift-gitops
spec:
  generators:
    - list:
        elements:
          - name: app-alpha
            namespace: app-alpha
            chartVersion: "0.1.0"
            repoURL: https://<owner>.github.io/<repo>
            chart: openshift-application
            values:
              name: app-alpha
              replicaCount: 2
          - name: app-beta
            namespace: app-beta
            chartVersion: "0.1.0"
            repoURL: https://<owner>.github.io/<repo>
            chart: openshift-application
            values:
              name: app-beta
              replicaCount: 1
  template:
    metadata:
      name: '{{name}}-application'
    spec:
      project: default
      source:
        repoURL: '{{repoURL}}'
        chart: '{{chart}}'
        targetRevision: '{{chartVersion}}'
        helm:
          values: |
            # inline values block rendered per element
            name: {{values.name}}
            replicaCount: {{values.replicaCount}}
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

Notes on values in the template
- The `helm.values` field in the template above is an inline YAML block. It allows you to set or override Helm chart values per generated Application. Alternatively, if your values are stored in Git you can use `source.repoURL` pointing to the Git repo and `path` to the value files.

3.2 Example: multiple sources (Git + Helm repo)
You can mix generators and elements that point to different sources. For example, use some elements whose source is a Helm repo and others whose source is a Git repo that contains a values file or a sub-chart.

Example element using a Git repo for values:
```yaml
# element example (inside list)
- name: app-git-values
  namespace: app-git
  repoURL: https://github.com/your-org/your-values-repo.git
  path: charts/overrides
  chart: openshift-application
  chartVersion: 0.1.0
```
In that case the `template.spec.source` would use `repoURL` and `path` for Git, and the `chart` would reference the hosted Helm chart (or you can include the chart package directly in that Git repo).

4) Useful workflows & tips
- Keep a `values/` directory in a centralized repo (or per-application folder) so you can reference `path` in the `Application` source and use `valueFiles` in Argo to load those per-app overrides.
- For Helm repos hosted on GitHub Pages, prefer absolute URLs in your `docs/index.yaml` so tooling that expects full URLs works reliably.
- Use `selfHeal` and `prune` with caution in production; they make Argo automatically correct drift and delete resources that are no longer in Git.
- Use `syncPolicy.automated.allowEmpty` if some apps occasionally have no resources and you want Argo to continue without error.

5) Example end-to-end commands
```bash
# package and index (from repo root)
cd generic-application-helm-chart-openshift
helm package . -d ../docs
helm repo index ../docs --url https://<owner>.github.io/<repo>

# commit & push
git add docs
git commit -m "Publish helm chart to docs for GitHub Pages"
git push origin main

# add repo locally and verify
helm repo add samplechart https://<owner>.github.io/<repo>
helm repo update
helm search repo samplechart/openshift-application --versions
```

6) Troubleshooting
- If `helm repo add` returns 404 for `/index.yaml`:
  - Confirm Files are in `docs/` and pushed to `main`.
  - Confirm Pages is configured to serve from `main` → `/docs`.
  - Make sure `index.yaml` `urls:` entries point to accessible `.tgz` files (absolute URLs are safest for Pages).
- If the ApplicationSet fails to generate Applications, check: Argo's logs, CRD versions, and the rendered template (Argo UI shows generated Applications and rendered manifests).

7) Where to go from here
- Automate packaging and publishing via CI (a GitHub Actions workflow can build the chart, update `docs/index.yaml`, and push changes — a workflow template is included in this repo).
- Use an ApplicationSet generator that best fits your workflow (`git`, `list`, `matrix`, `clusters`).
- Secure your Argo installations with RBAC and GPG-signed commits where appropriate.

If you'd like, I can:
- regenerate the `docs/index.yaml` with the exact GitHub Pages URL for your repo and commit it, or
- create a sample `ApplicationSet` YAML for the exact app names and namespaces you want to deploy, or
- walk through enabling GitHub Pages and testing `helm repo add` step-by-step interactively.
