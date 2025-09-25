# Kibee FastAPI Demo

Application FastAPI minimaliste servant de point de départ pour un déploiement sur Azure Container Apps.

## Prérequis

- Python 3.11+
- [pip](https://pip.pypa.io/en/stable/)
- [Docker](https://docs.docker.com/get-docker/) (pour l'exécution conteneurisée)
- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) 2.45+ avec l'extension `containerapp`

## Installation locale

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload
```

L'application est disponible sur http://127.0.0.1:8000.

## Exécution via Docker

```bash
docker build -t kibee-app:latest .
docker run -p 8000:8000 kibee-app:latest
```

## Déploiement sur Azure Container Apps

Variables d'environnement utiles :

```bash
RESOURCE_GROUP=kibee-group
LOCATION=westeurope
ACR_NAME=kibeeacr$RANDOM
APP_NAME=kibee-app
ENV_NAME=kibee-env
IMAGE_TAG=latest
```

1. Création des ressources :
   ```bash
   az login
   az group create --name $RESOURCE_GROUP --location $LOCATION
   az monitor log-analytics workspace create \
     --resource-group $RESOURCE_GROUP --workspace-name ${ENV_NAME}-logs --location $LOCATION
   az containerapp env create \
     --name $ENV_NAME \
     --resource-group $RESOURCE_GROUP \
     --location $LOCATION \
     --logs-workspace-id $(az monitor log-analytics workspace show -g $RESOURCE_GROUP -n ${ENV_NAME}-logs --query customerId -o tsv) \
     --logs-workspace-key $(az monitor log-analytics workspace get-shared-keys -g $RESOURCE_GROUP -n ${ENV_NAME}-logs --query primarySharedKey -o tsv)
   az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic --admin-enabled true
   ```

2. Build & push de l'image :
   ```bash
   az acr build --registry $ACR_NAME --image $APP_NAME:$IMAGE_TAG .
   ```

3. Déploiement dans ACA :
   ```bash
   ACR_SERVER=$(az acr show --name $ACR_NAME --query loginServer -o tsv)
   ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query username -o tsv)
   ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

   az containerapp create \
     --name $APP_NAME \
     --resource-group $RESOURCE_GROUP \
     --environment $ENV_NAME \
     --image $ACR_SERVER/$APP_NAME:$IMAGE_TAG \
     --target-port 8000 \
     --ingress external \
     --registry-server $ACR_SERVER \
     --registry-username $ACR_USERNAME \
     --registry-password $ACR_PASSWORD \
     --cpu 0.5 --memory 1.0Gi
   ```

4. Récupérer l'URL publique :
   ```bash
   az containerapp show --name $APP_NAME --resource-group $RESOURCE_GROUP --query properties.configuration.ingress.fqdn -o tsv
   ```

## Tests

Les endpoints peuvent être testés avec `curl` :

```bash
curl http://127.0.0.1:8000/
```

Pour la documentation interactive, consultez http://127.0.0.1:8000/docs.

## License

MIT
