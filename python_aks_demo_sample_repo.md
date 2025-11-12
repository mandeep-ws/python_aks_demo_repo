# python-aks-demo (sample repo)

This canvas contains a complete sample repository you can clone/use for an enterprise end-to-end CI/CD hands-on lab: build a Python package, run unit tests, static analysis hooks, multi-stage Docker build, Trivy scan, push to ACR, ephemeral integration test deployment to AKS via Helm, and staging/prod deployments. Replace service-connection names and Azure resource names with your environment values.

---

## Repository layout

```
python-aks-demo/
├─ azure-pipelines.yml
├─ Dockerfile
├─ pyproject.toml
├─ requirements.txt
├─ src/
│  └─ mypkg/
│     ├─ __init__.py
│     └─ app.py
├─ tests/
│  ├─ unit/
│  │  └─ test_app.py
│  └─ integration/
│     └─ test_integration.py
├─ helm/
│  └─ myapp/
│     ├─ Chart.yaml
│     ├─ values.yaml
│     └─ templates/
│        ├─ deployment.yaml
│        └─ service.yaml
└─ sonar-project.properties
```

---

## 1) `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=61", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "mypkg"
version = "0.1.0"
description = "Sample Python package for AKS CI/CD lab"
authors = [ { name = "Demo Team" } ]

[tool.pytest.ini_options]
minversion = "6.0"
addopts = "-q --disable-warnings --maxfail=1"

[tool.coverage]
run = { branch = true }
```

---

## 2) `requirements.txt`

```
flask==2.3.2
pytest==7.4.0
requests==2.31.0
pytest-cov==4.1.0
coverage==7.2.7
```

---

## 3) `src/mypkg/__init__.py`

```python
from .app import create_app

__all__ = ["create_app"]
```

---

## 4) `src/mypkg/app.py`

```python
from flask import Flask, jsonify


def create_app():
    app = Flask(__name__)

    @app.route("/health")
    def health():
        return jsonify({"status": "ok"}), 200

    @app.route("/hello")
    def hello():
        return jsonify({"message": "Hello from mypkg"}), 200

    return app


if __name__ == '__main__':
    create_app().run(host='0.0.0.0', port=5000)
```

---

## 5) `tests/unit/test_app.py`

```python
from mypkg.app import create_app


def test_health():
    app = create_app()
    client = app.test_client()
    resp = client.get('/health')
    assert resp.status_code == 200
    assert resp.get_json() == {'status': 'ok'}


def test_hello():
    app = create_app()
    client = app.test_client()
    resp = client.get('/hello')
    assert resp.status_code == 200
    assert 'Hello' in resp.get_json()['message']
```

---

## 6) `tests/integration/test_integration.py` (simple integration example)

```python
import requests
import os


def test_hello_endpoint():
    # CI deploys the app and sets BASE_URL env var for integration stage
    base = os.getenv('BASE_URL', 'http://localhost:5000')
    r = requests.get(f"{base}/hello", timeout=5)
    assert r.status_code == 200
    assert 'Hello' in r.json()['message']
```

---

## 7) `Dockerfile` (multi-stage)

```dockerfile
# ---- build stage ----
FROM python:3.11-slim AS builder
WORKDIR /src

# system deps (small image)
RUN apt-get update && apt-get install -y --no-install-recommends build-essential && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml requirements.txt ./
RUN python -m pip install --upgrade pip
RUN pip wheel --wheel-dir /wheels -r requirements.txt

COPY src/ ./src
RUN pip install --no-deps --no-index --find-links /wheels .

# ---- runtime stage ----
FROM python:3.11-slim
WORKDIR /app

# create non-root user
RUN useradd -m appuser

COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY src/ ./src

USER appuser
ENV PYTHONUNBUFFERED=1
EXPOSE 5000
CMD ["python", "-m", "mypkg.app"]
```

---

## 8) `helm/myapp/Chart.yaml`

```yaml
apiVersion: v2
name: myapp
description: Sample Helm chart for mypkg
type: application
version: 0.1.0
appVersion: "0.1.0"
```

---

## 9) `helm/myapp/values.yaml`

```yaml
replicaCount: 1

image:
  repository: myacr.azurecr.io/mypkg
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources: {}

livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## 10) `helm/myapp/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    app: {{ include "myapp.name" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "myapp.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "myapp.name" . }}
    spec:
      containers:
        - name: {{ include "myapp.name" . }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 5000
          livenessProbe:
            httpGet:
              path: "/health"
              port: 5000
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
          readinessProbe:
            httpGet:
              path: "/health"
              port: 5000
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
```

---

## 11) `helm/myapp/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ include "myapp.name" . }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 5000
```

---

## 12) `azure-pipelines.yml` (opinionated multi-stage pipeline)

```yaml
trigger:
  branches:
    include:
      - main
      - release/*

variables:
  imageName: 'myacr.azurecr.io/mypkg'
  tag: '$(Build.BuildId)'
  azureSubscription: 'AzureRM-ServiceConnection'     # replace
  aksClusterName: 'myaks'                            # replace
  aksResourceGroup: 'rg-aks'                        # replace
  sonarService: 'SonarQubeServiceConnection'        # replace
  acrServiceConnection: 'ACR-ServiceConnection'     # replace

stages:
- stage: Build
  displayName: Build & Test
  jobs:
  - job: UnitAndSonar
    pool: { vmImage: 'ubuntu-latest' }
    steps:
      - checkout: self
      - task: UsePythonVersion@0
        inputs:
          versionSpec: '3.11'
      - script: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pytest tests/unit --junitxml=test-results/junit.xml --cov=src/mypkg --cov-report=xml:coverage.xml
        displayName: Run unit tests
      - task: PublishTestResults@2
        inputs:
          testResultsFiles: 'test-results/junit.xml'

      - task: SonarQubePrepare@5
        inputs:
          SonarQube: $(sonarService)
          scannerMode: 'CLI'
          cliProjectKey: 'mypkg'
          extraProperties: |
            sonar.python.coverage.reportPaths=coverage.xml
      - task: SonarQubeAnalyze@5
      - task: SonarQubePublish@5
        inputs:
          pollingTimeoutSec: '300'

- stage: Image
  displayName: Build Image & Scan
  dependsOn: Build
  jobs:
  - job: BuildAndScan
    pool: { vmImage: 'ubuntu-latest' }
    steps:
      - checkout: self
      - task: Docker@2
        displayName: Build image
        inputs:
          command: build
          Dockerfile: Dockerfile
          tags: |
            $(tag)
          containerRegistry: $(acrServiceConnection)
          repository: 'mypkg'

      - script: |
          echo "Install trivy"
          curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b $(Agent.ToolsDirectory)/trivy
          $(Agent.ToolsDirectory)/trivy image --format json --output trivy.json --exit-code 1 --severity CRITICAL,HIGH $(imageName):$(tag) || true
        displayName: Trivy scan

      - task: Docker@2
        displayName: Push image
        inputs:
          command: push
          containerRegistry: $(acrServiceConnection)
          repository: 'mypkg'
          tags: |
            $(tag)

      - task: PublishBuildArtifacts@1
        inputs:
          pathToPublish: 'trivy.json'
          artifactName: 'security-scan'

- stage: IntegrationTest
  displayName: Integration Tests
  dependsOn: Image
  jobs:
  - job: RunIntegration
    pool: { vmImage: 'ubuntu-latest' }
    steps:
      - checkout: self
      - task: AzureCLI@2
        inputs:
          azureSubscription: $(azureSubscription)
          scriptType: bash
          scriptLocation: inlineScript
          inlineScript: |
            az aks get-credentials --name $(aksClusterName) --resource-group $(aksResourceGroup) --admin
            ns=integration-$(Build.BuildId)
            kubectl create namespace $ns || true
            helm upgrade --install myapp-test helm/myapp --namespace $ns --set image.repository=$(imageName) --set image.tag=$(tag)
            kubectl rollout status deployment/myapp-test --namespace $ns --timeout=120s || true
            # optional: get service IP (if using LoadBalancer)
            # for simple labs we rely on port-forwarding below
      - script: |
          echo "Port-forwarding service and running integration tests"
          kubectl port-forward --namespace integration-$(Build.BuildId) svc/myapp-test 5000:80 &
          sleep 3
          export BASE_URL=http://localhost:5000
          pytest tests/integration --junitxml=integration-results/junit.xml
        displayName: Run integration tests
      - task: PublishTestResults@2
        inputs:
          testResultsFiles: 'integration-results/junit.xml'
      - task: AzureCLI@2
        inputs:
          azureSubscription: $(azureSubscription)
          scriptType: bash
          scriptLocation: inlineScript
          inlineScript: |
            ns=integration-$(Build.BuildId)
            helm uninstall myapp-test --namespace $ns || true
            kubectl delete namespace $ns || true

- stage: DeployStaging
  displayName: Deploy to Staging
  dependsOn: IntegrationTest
  jobs:
  - deployment: StagingDeploy
    environment: 'staging'
    strategy:
      runOnce:
        deploy:
          steps:
            - task: AzureCLI@2
              inputs:
                azureSubscription: $(azureSubscription)
                scriptType: bash
                scriptLocation: inlineScript
                inlineScript: |
                  az aks get-credentials --name $(aksClusterName) --resource-group $(aksResourceGroup)
                  helm upgrade --install myapp ./helm/myapp --namespace staging --create-namespace --set image.repository=$(imageName) --set image.tag=$(tag)

- stage: DeployProd
  displayName: Deploy to Production
  dependsOn: DeployStaging
  approval:
    approvals:
      - approver: 'sre-team@contoso.com' # configure environment approvals in Azure DevOps
  jobs:
  - deployment: ProductionDeploy
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
            - task: AzureCLI@2
              inputs:
                azureSubscription: $(azureSubscription)
                scriptType: bash
                scriptLocation: inlineScript
                inlineScript: |
                  az aks get-credentials --name $(aksClusterName) --resource-group $(aksResourceGroup)
                  helm upgrade --install myapp ./helm/myapp --namespace prod --create-namespace --set image.repository=$(imageName) --set image.tag=$(tag)
```

---

## 13) `sonar-project.properties` (optional)

```properties
sonar.projectKey=mypkg
sonar.projectName=mypkg
sonar.sources=src
sonar.python.coverage.reportPaths=coverage.xml
```

---

### How to use this sample

1. Create an Azure Container Registry and an AKS cluster (or reuse existing). Note/update the names in `azure-pipelines.yml` variables.
2. Create Azure DevOps service connections:
   - Azure Resource Manager (for AKS)
   - Docker/ACR service connection (for Docker@2)
   - SonarQube service connection (if you run SonarQube)
3. Push this repo to your Azure DevOps Git repository and create a pipeline using `azure-pipelines.yml`.
4. Run pipeline; inspect build artifacts (test results, Trivy output). For local testing you can run unit tests with `pytest tests/unit` and build the Docker image locally.

---

If you want, I can now:
- generate the repository as a downloadable ZIP file, or
- create a runnable Terraform/az cli script to provision ACR + AKS and the required service principals/service connections, or
- adapt the repo to use GitHub Actions instead of Azure Pipelines.

Which next step would you like?

