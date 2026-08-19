# 1. Práctica 1. Preparar el despliegue de una solución GenAI utilizando Azure AI Foundry, Azure OpenAI Service y Azure Container Apps, aplicando buenas prácticas de seguridad y administración de secretos.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 45 minutos |
| Complejidad | Media |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica prepararás la plataforma Azure compartida para desplegar una solución GenAI segura. Crearás recursos de Azure AI Foundry, Azure OpenAI Service, Azure Key Vault, Azure Container Registry, Log Analytics y Azure Container Apps.

La práctica aplica el principio de mínimo privilegio mediante identidad administrada y RBAC. No se incluirán claves de Azure OpenAI en código, imágenes Docker, comandos que muestren su valor, archivos versionados ni configuraciones públicas.

> **Resultado esperado de la práctica:** una Container App base con identidad administrada, permisos mínimos sobre Key Vault y Azure OpenAI, una referencia segura a un secreto de Key Vault y un inventario local de recursos requerido para la práctica `07-00-02`.

## Objetivos de aprendizaje

Al finalizar la práctica, podrás:

- [ ] Crear un grupo de recursos y registrar los proveedores necesarios para una solución GenAI en East US 2.
- [ ] Crear un hub y proyecto de Azure AI Foundry, además de un recurso Azure OpenAI con el modelo `gpt-4o-mini` versión `2024-07-18`.
- [ ] Proteger la clave de Azure OpenAI en Azure Key Vault sin persistirla en archivos de código o configuración versionados.
- [ ] Crear Azure Container Registry, un entorno de Azure Container Apps y una Container App con identidad administrada por el sistema.
- [ ] Configurar RBAC mínimo y validar la topología de seguridad mediante Azure CLI y la API REST de Azure Resource Manager.

## Prerrequisitos

### Conocimientos requeridos

- Conocimientos intermedios de Python y consumo de APIs REST.
- Conocimiento básico de contenedores, variables de entorno y servicios de Azure.
- Comprensión de conceptos de IA generativa, modelos de chat y despliegues de modelos.
- Conocimiento general de Azure RBAC e identidades administradas.

### Acceso y permisos requeridos

Necesitas:

- Una suscripción activa de Azure.
- Rol **Contributor** sobre la suscripción o sobre el grupo de recursos objetivo.
- Rol **User Access Administrator** si debes crear asignaciones RBAC por tu cuenta.
- Acceso aprobado a Azure OpenAI.
- Cuota disponible para `gpt-4o-mini`, versión `2024-07-18`, en **East US 2**.
- Azure CLI `2.67.0` o una versión compatible.
- Docker Desktop `4.37.1` o Docker Engine `27.4.1` instalado y en ejecución. En esta práctica no construirás una imagen propia, pero Docker será necesario en la siguiente práctica.

> **Importante:** si tu organización no dispone de cuota en East US 2, solicita al instructor una región alternativa aprobada que tenga disponible exactamente el modelo y versión requeridos. No cambies la región sin autorización.

## Entorno del laboratorio

### Recursos que se crearán

| Tipo de recurso | Nombre requerido |
|---|---|
| Grupo de recursos | `rg-genai-agents-eastus2` |
| Hub de Azure AI Foundry | `genai-agents-hub` |
| Proyecto de Azure AI Foundry | `genai-agents-project` |
| Recurso Azure OpenAI | `aoai-genai-agents-<sufijo>` |
| Despliegue de modelo | `genai-chat-model` |
| Azure Key Vault | `kv-genai-agents-<sufijo>` |
| Azure Container Registry | `acrgenaiagents<sufijo>` |
| Log Analytics Workspace | `law-genai-agents-<sufijo>` |
| Entorno de Container Apps | `cae-genai-agents` |
| Container App | `ca-genai-agent-api` |

### Convenciones de seguridad

| Elemento | Política aplicada |
|---|---|
| Clave de Azure OpenAI | Se almacena exclusivamente en Azure Key Vault. |
| Código fuente | No contiene claves, endpoints sensibles ni cadenas de conexión. |
| Archivo `.env` | No se versiona; se ignora mediante `.gitignore`. |
| Archivo `.env.example` | Solo documenta nombres de variables, nunca valores secretos. |
| Container App | Usa identidad administrada asignada por el sistema. |
| Azure OpenAI | Se prepara para autenticación con Microsoft Entra ID mediante RBAC. |
| Key Vault | Usa el modelo de autorización RBAC, no políticas de acceso heredadas. |
| Ingreso de Container App | Se configura inicialmente como interno. |

### Preparación de la terminal

Abre una terminal y autentícate en Azure:

```bash
az login
az account show --output table
```

Si tienes más de una suscripción, selecciona explícitamente la suscripción del laboratorio:

```bash
az account set --subscription "<ID-O-NOMBRE-DE-SUSCRIPCION>"
az account show --query "{subscription:name,id:id,tenantId:tenantId}" --output json
```

Crea variables de sesión. El sufijo se deriva de la suscripción para reducir colisiones de nombres globales.

```bash
export LOCATION="eastus2"
export RG="rg-genai-agents-eastus2"

export SUFFIX="$(az account show --query id --output tsv | tr -d '-' | cut -c1-8)"

export FOUNDRY_HUB="genai-agents-hub"
export FOUNDRY_PROJECT="genai-agents-project"
export AOAI_NAME="aoai-genai-agents-${SUFFIX}"
export KV_NAME="kv-genai-agents-${SUFFIX}"
export ACR_NAME="acrgenaiagents${SUFFIX}"
export LAW_NAME="law-genai-agents-${SUFFIX}"
export CAE_NAME="cae-genai-agents"
export CA_NAME="ca-genai-agent-api"

export MODEL_NAME="gpt-4o-mini"
export MODEL_VERSION="2024-07-18"
export DEPLOYMENT_NAME="genai-chat-model"
export OPENAI_API_VERSION="2024-10-21"
```

Confirma los nombres calculados:

```bash
printf "Grupo de recursos: %s\n" "$RG"
printf "Azure OpenAI: %s\n" "$AOAI_NAME"
printf "Key Vault: %s\n" "$KV_NAME"
printf "ACR: %s\n" "$ACR_NAME"
printf "Sufijo: %s\n" "$SUFFIX"
```

> **Nota:** Azure Key Vault permite nombres de entre 3 y 24 caracteres, mientras que Azure Container Registry permite únicamente letras minúsculas y números. Los nombres calculados cumplen estas restricciones.

---

## Procedimiento paso a paso

### Paso 1. Crear el grupo de recursos y registrar proveedores

**Objetivo:** crear el contenedor lógico de recursos del laboratorio y habilitar los proveedores de Azure requeridos.

#### Instrucciones

1. Crea el grupo de recursos en East US 2:

   ```bash
   az group create \
     --name "$RG" \
     --location "$LOCATION"
   ```

2. Registra los proveedores requeridos por la práctica:

   ```bash
   for PROVIDER in \
     Microsoft.App \
     Microsoft.OperationalInsights \
     Microsoft.CognitiveServices \
     Microsoft.KeyVault \
     Microsoft.ContainerRegistry \
     Microsoft.MachineLearningServices
   do
     az provider register --namespace "$PROVIDER"
   done
   ```

3. Consulta el estado de registro de los proveedores:

   ```bash
   az provider show \
     --namespace Microsoft.App \
     --query registrationState \
     --output tsv

   az provider show \
     --namespace Microsoft.CognitiveServices \
     --query registrationState \
     --output tsv

   az provider show \
     --namespace Microsoft.KeyVault \
     --query registrationState \
     --output tsv
   ```

4. Si alguno aparece como `Registering`, espera unos minutos y vuelve a consultar:

   ```bash
   az provider list \
     --query "[?registrationState!='Registered'].{Proveedor:namespace,Estado:registrationState}" \
     --output table
   ```

#### Resultado esperado

Se crea el grupo de recursos `rg-genai-agents-eastus2` en `eastus2`. Los proveedores deben terminar en estado `Registered`.

#### Verificación

```bash
az group show \
  --name "$RG" \
  --query "{nombre:name,ubicacion:location,estado:properties.provisioningState}" \
  --output table
```

La salida debe incluir:

```text
Nombre                         Ubicacion    Estado
-----------------------------  -----------  ---------
rg-genai-agents-eastus2        eastus2      Succeeded
```

---

### Paso 2. Crear el hub y el proyecto de Azure AI Foundry

**Objetivo:** crear el espacio de trabajo de Azure AI Foundry que organizará los activos y proyectos de IA del lote.

#### Instrucciones

1. Abre [Azure AI Foundry](https://ai.azure.com/) e inicia sesión con la misma cuenta utilizada en Azure CLI.

2. Selecciona **Management center** o **Centro de administración**.

3. En la sección de hubs, selecciona **New hub** o **Nuevo hub**.

4. Configura el hub con los siguientes valores:

   | Campo | Valor |
   |---|---|
   | Nombre | `genai-agents-hub` |
   | Suscripción | La suscripción seleccionada para el laboratorio |
   | Grupo de recursos | `rg-genai-agents-eastus2` |
   | Ubicación | `East US 2` |

5. Revisa los recursos asociados que Azure AI Foundry pueda crear para el hub, como almacenamiento, Key Vault interno, Application Insights o Azure AI Services. Estos recursos administrados por Foundry no sustituyen al Key Vault explícito que crearás en el siguiente paso.

6. Crea el hub y espera a que finalice el aprovisionamiento.

7. Dentro del hub creado, selecciona **New project** o **Nuevo proyecto**.

8. Configura el proyecto:

   | Campo | Valor |
   |---|---|
   | Nombre | `genai-agents-project` |
   | Hub | `genai-agents-hub` |
   | Grupo de recursos | `rg-genai-agents-eastus2` |

9. Crea el proyecto.

> **Nota de compatibilidad:** la interfaz de Azure AI Foundry evoluciona con frecuencia. Si la interfaz solicita crear un proyecto antes que un hub, crea el proyecto `genai-agents-project` y selecciona o crea el hub `genai-agents-hub` durante el asistente. Conserva los nombres, grupo de recursos y región definidos en esta práctica.

#### Resultado esperado

Existe un hub denominado `genai-agents-hub` y un proyecto denominado `genai-agents-project` asociado a ese hub.

#### Verificación

En Azure AI Foundry:

1. Abre el proyecto `genai-agents-project`.
2. Confirma que el nombre del proyecto aparece en el selector superior.
3. Abre la sección de administración del proyecto.
4. Verifica que el hub asociado es `genai-agents-hub`.

Como verificación adicional desde Azure Portal, revisa que los recursos relacionados se encuentren en `rg-genai-agents-eastus2`.

---

### Paso 3. Crear Azure OpenAI y desplegar el modelo requerido

**Objetivo:** crear el recurso Azure OpenAI y desplegar exactamente `gpt-4o-mini` versión `2024-07-18` con el nombre `genai-chat-model`.

#### Instrucciones

1. Crea el recurso Azure OpenAI:

   ```bash
   az cognitiveservices account create \
     --name "$AOAI_NAME" \
     --resource-group "$RG" \
     --location "$LOCATION" \
     --kind OpenAI \
     --sku S0 \
     --yes
   ```

2. Obtén el endpoint del recurso sin consultar ni mostrar claves:

   ```bash
   export AZURE_OPENAI_ENDPOINT="$(az cognitiveservices account show \
     --name "$AOAI_NAME" \
     --resource-group "$RG" \
     --query properties.endpoint \
     --output tsv)"

   printf "%s\n" "$AZURE_OPENAI_ENDPOINT"
   ```

3. Crea el despliegue del modelo:

   ```bash
   az cognitiveservices account deployment create \
     --name "$AOAI_NAME" \
     --resource-group "$RG" \
     --deployment-name "$DEPLOYMENT_NAME" \
     --model-name "$MODEL_NAME" \
     --model-version "$MODEL_VERSION" \
     --model-format OpenAI \
     --sku-name "Standard" \
     --sku-capacity 1
   ```

4. Consulta el estado del despliegue:

   ```bash
   az cognitiveservices account deployment show \
     --name "$AOAI_NAME" \
     --resource-group "$RG" \
     --deployment-name "$DEPLOYMENT_NAME" \
     --query "{nombre:name,estado:properties.provisioningState,modelo:properties.model.name,version:properties.model.version}" \
     --output table
   ```

> **Importante:** no cambies el nombre del despliegue. La práctica `07-00-02` utilizará obligatoriamente `genai-chat-model`.

#### Resultado esperado

Se crea el recurso Azure OpenAI y el despliegue muestra:

- Modelo: `gpt-4o-mini`
- Versión: `2024-07-18`
- Nombre del despliegue: `genai-chat-model`
- Estado: `Succeeded`

#### Verificación

Ejecuta:

```bash
az cognitiveservices account deployment list \
  --name "$AOAI_NAME" \
  --resource-group "$RG" \
  --query "[].{Nombre:name,Modelo:properties.model.name,Version:properties.model.version,Estado:properties.provisioningState}" \
  --output table
```

También puedes validar la disponibilidad desde Azure AI Foundry:

1. Abre `genai-agents-project`.
2. Accede a **Models + endpoints**.
3. Confirma que aparece el despliegue `genai-chat-model`.
4. Confirma que el modelo corresponde a `gpt-4o-mini`, versión `2024-07-18`.

---

### Paso 4. Crear Azure Key Vault y proteger la clave de Azure OpenAI

**Objetivo:** crear un almacén de secretos con autorización RBAC y guardar la clave de Azure OpenAI sin exponerla en archivos locales.

#### Instrucciones

1. Crea Azure Key Vault habilitando autorización mediante Azure RBAC:

   ```bash
   az keyvault create \
     --name "$KV_NAME" \
     --resource-group "$RG" \
     --location "$LOCATION" \
     --enable-rbac-authorization true \
     --sku standard
   ```

2. Obtén el ID del Key Vault:

   ```bash
   export KV_ID="$(az keyvault show \
     --name "$KV_NAME" \
     --resource-group "$RG" \
     --query id \
     --output tsv)"
   ```

3. Guarda la primera clave de Azure OpenAI directamente en Key Vault.

   El siguiente comando no imprime el valor de la clave. Evita ejecutar comandos de consulta de claves con salida `json`, `table` o `tsv` en pantalla.

   ```bash
   az keyvault secret set \
     --vault-name "$KV_NAME" \
     --name "azure-openai-api-key" \
     --value "$(az cognitiveservices account keys list \
       --name "$AOAI_NAME" \
       --resource-group "$RG" \
       --query key1 \
       --output tsv)" \
     --output none
   ```

4. Recupera solo el identificador URI del secreto, no su valor:

   ```bash
   export KV_SECRET_URI="$(az keyvault secret show \
     --vault-name "$KV_NAME" \
     --name "azure-openai-api-key" \
     --query id \
     --output tsv)"

   printf "%s\n" "$KV_SECRET_URI"
   ```

5. Confirma que el secreto existe sin mostrar su contenido:

   ```bash
   az keyvault secret list \
     --vault-name "$KV_NAME" \
     --query "[].{Nombre:name,Id:id,Enabled:attributes.enabled}" \
     --output table
   ```

#### Resultado esperado

Existe un Key Vault con autorización RBAC habilitada y un secreto llamado `azure-openai-api-key`. La terminal no debe mostrar el valor de la clave.

#### Verificación

```bash
az keyvault show \
  --name "$KV_NAME" \
  --query "{nombre:name,rbacHabilitado:properties.enableRbacAuthorization,uri:properties.vaultUri}" \
  --output json
```

La propiedad `rbacHabilitado` debe ser `true`.

---

### Paso 5. Crear ACR, Log Analytics y el entorno de Azure Container Apps

**Objetivo:** crear los recursos de ejecución, registro de imágenes y observabilidad básica para el servicio integrador.

#### Instrucciones

1. Crea Azure Container Registry con SKU Basic:

   ```bash
   az acr create \
     --name "$ACR_NAME" \
     --resource-group "$RG" \
     --location "$LOCATION" \
     --sku Basic
   ```

2. Obtén el servidor de inicio de sesión del registro:

   ```bash
   export ACR_LOGIN_SERVER="$(az acr show \
     --name "$ACR_NAME" \
     --resource-group "$RG" \
     --query loginServer \
     --output tsv)"

   printf "%s\n" "$ACR_LOGIN_SERVER"
   ```

3. Crea un espacio de trabajo de Log Analytics:

   ```bash
   az monitor log-analytics workspace create \
     --workspace-name "$LAW_NAME" \
     --resource-group "$RG" \
     --location "$LOCATION"
   ```

4. Obtén el identificador y una clave compartida del espacio de trabajo. La clave se usará únicamente para crear el entorno de Container Apps y no se guardará en archivos.

   ```bash
   export LAW_ID="$(az monitor log-analytics workspace show \
     --workspace-name "$LAW_NAME" \
     --resource-group "$RG" \
     --query customerId \
     --output tsv)"

   export LAW_KEY="$(az monitor log-analytics workspace get-shared-keys \
     --workspace-name "$LAW_NAME" \
     --resource-group "$RG" \
     --query primarySharedKey \
     --output tsv)"
   ```

5. Crea el entorno administrado de Azure Container Apps:

   ```bash
   az containerapp env create \
     --name "$CAE_NAME" \
     --resource-group "$RG" \
     --location "$LOCATION" \
     --logs-workspace-id "$LAW_ID" \
     --logs-workspace-key "$LAW_KEY"
   ```

6. Elimina la variable sensible de la sesión cuando el entorno haya sido creado:

   ```bash
   unset LAW_KEY
   ```

#### Resultado esperado

Existen un Azure Container Registry, un Log Analytics Workspace y un entorno administrado de Container Apps con registros centralizados.

#### Verificación

```bash
az containerapp env show \
  --name "$CAE_NAME" \
  --resource-group "$RG" \
  --query "{nombre:name,ubicacion:location,estado:properties.provisioningState}" \
  --output table
```

La propiedad `estado` debe indicar `Succeeded`.

---

### Paso 6. Crear la Container App base con identidad administrada

**Objetivo:** crear una Container App inicial con ingreso interno e identidad administrada asignada por el sistema.

#### Instrucciones

1. Crea una aplicación base usando una imagen pública mínima. Esta imagen es temporal y será reemplazada por la API FastAPI contenerizada en la práctica `07-00-02`.

   ```bash
   az containerapp create \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --environment "$CAE_NAME" \
     --image "mcr.microsoft.com/k8se/quickstart:latest" \
     --ingress internal \
     --target-port 80 \
     --min-replicas 1 \
     --max-replicas 1 \
     --system-assigned
   ```

2. Obtén el identificador principal de la identidad administrada:

   ```bash
   export CA_PRINCIPAL_ID="$(az containerapp show \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --query identity.principalId \
     --output tsv)"

   export CA_RESOURCE_ID="$(az containerapp show \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --query id \
     --output tsv)"

   printf "Principal ID: %s\n" "$CA_PRINCIPAL_ID"
   ```

3. Comprueba el tipo de identidad asignada:

   ```bash
   az containerapp show \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --query "{nombre:name,tipoIdentidad:identity.type,principalId:identity.principalId,estado:properties.provisioningState}" \
     --output json
   ```

#### Resultado esperado

La aplicación `ca-genai-agent-api` existe, tiene ingreso interno y muestra una identidad de tipo `SystemAssigned`.

#### Verificación

```bash
az containerapp show \
  --name "$CA_NAME" \
  --resource-group "$RG" \
  --query "{nombre:name,ingress:properties.configuration.ingress.external,identidad:identity.type}" \
  --output table
```

La columna `ingress` debe mostrar `False` y la identidad debe mostrar `SystemAssigned`.

---

### Paso 7. Asignar RBAC mínimo a la identidad administrada

**Objetivo:** permitir que la Container App lea secretos de Key Vault y consuma Azure OpenAI mediante Microsoft Entra ID, sin otorgar permisos excesivos.

#### Instrucciones

1. Obtén el ID del recurso Azure OpenAI:

   ```bash
   export AOAI_ID="$(az cognitiveservices account show \
     --name "$AOAI_NAME" \
     --resource-group "$RG" \
     --query id \
     --output tsv)"
   ```

2. Asigna el rol **Key Vault Secrets User** sobre el Key Vault:

   ```bash
   az role assignment create \
     --assignee-object-id "$CA_PRINCIPAL_ID" \
     --assignee-principal-type ServicePrincipal \
     --role "Key Vault Secrets User" \
     --scope "$KV_ID"
   ```

3. Asigna el rol **Cognitive Services OpenAI User** sobre el recurso Azure OpenAI:

   ```bash
   az role assignment create \
     --assignee-object-id "$CA_PRINCIPAL_ID" \
     --assignee-principal-type ServicePrincipal \
     --role "Cognitive Services OpenAI User" \
     --scope "$AOAI_ID"
   ```

4. Como preparación para que la Container App obtenga imágenes desde ACR en la siguiente práctica, asigna el rol mínimo `AcrPull` sobre el registro:

   ```bash
   export ACR_ID="$(az acr show \
     --name "$ACR_NAME" \
     --resource-group "$RG" \
     --query id \
     --output tsv)"

   az role assignment create \
     --assignee-object-id "$CA_PRINCIPAL_ID" \
     --assignee-principal-type ServicePrincipal \
     --role "AcrPull" \
     --scope "$ACR_ID"
   ```

5. Consulta las asignaciones RBAC de la identidad:

   ```bash
   az role assignment list \
     --assignee-object-id "$CA_PRINCIPAL_ID" \
     --all \
     --query "[].{Rol:roleDefinitionName,Scope:scope}" \
     --output table
   ```

> **Nota:** la propagación de asignaciones RBAC puede tardar entre varios segundos y algunos minutos. Este retraso es normal en Microsoft Entra ID y Azure RBAC.

#### Resultado esperado

La identidad administrada de `ca-genai-agent-api` dispone de los siguientes roles mínimos:

| Rol | Ámbito |
|---|---|
| `Key Vault Secrets User` | Key Vault del laboratorio |
| `Cognitive Services OpenAI User` | Recurso Azure OpenAI del laboratorio |
| `AcrPull` | Azure Container Registry del laboratorio |

#### Verificación

Ejecuta el siguiente comando y revisa que se muestren exactamente los roles esperados:

```bash
az role assignment list \
  --assignee-object-id "$CA_PRINCIPAL_ID" \
  --all \
  --query "[].roleDefinitionName" \
  --output table
```

No otorgues roles amplios como `Owner`, `Contributor` o `Key Vault Administrator` a la identidad de la aplicación.

---

### Paso 8. Configurar variables no sensibles y la referencia de secreto de Key Vault

**Objetivo:** preparar la configuración de la Container App separando valores no sensibles de secretos.

#### Instrucciones

1. Configura las variables no sensibles requeridas por la futura API:

   ```bash
   az containerapp update \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --set-env-vars \
       "AZURE_OPENAI_ENDPOINT=$AZURE_OPENAI_ENDPOINT" \
       "AZURE_OPENAI_DEPLOYMENT_NAME=$DEPLOYMENT_NAME" \
       "AZURE_OPENAI_API_VERSION=$OPENAI_API_VERSION"
   ```

2. Configura una referencia segura al secreto de Key Vault. La sintaxis `keyvaultref:` guarda una referencia, no el valor del secreto.

   ```bash
   az containerapp secret set \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --secrets "azure-openai-api-key=keyvaultref:${KV_SECRET_URI},identityref:system"
   ```

3. Asocia el secreto referenciado a una variable de entorno sensible. El valor se resuelve durante la ejecución de la aplicación y no debe aparecer en el código.

   ```bash
   az containerapp update \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --set-env-vars \
       "AZURE_OPENAI_API_KEY=secretref:azure-openai-api-key"
   ```

4. Lista los secretos configurados. Azure CLI debe mostrar nombres y referencias, pero no valores:

   ```bash
   az containerapp secret list \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --output table
   ```

5. Consulta únicamente los nombres de variables de entorno configuradas:

   ```bash
   az containerapp show \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --query "properties.template.containers[0].env[].name" \
     --output table
   ```

#### Resultado esperado

La Container App contiene:

- `AZURE_OPENAI_ENDPOINT`
- `AZURE_OPENAI_DEPLOYMENT_NAME`
- `AZURE_OPENAI_API_VERSION`
- `AZURE_OPENAI_API_KEY`, referenciada mediante `secretref:azure-openai-api-key`

La clave real no aparece en la salida de los comandos, en el repositorio ni en archivos locales.

#### Verificación

Ejecuta:

```bash
az containerapp show \
  --name "$CA_NAME" \
  --resource-group "$RG" \
  --query "properties.template.containers[0].env" \
  --output json
```

Verifica lo siguiente:

- Las tres variables no sensibles incluyen un campo `value`.
- La variable `AZURE_OPENAI_API_KEY` incluye un campo `secretRef`.
- No existe un valor de clave visible en la salida.

---

### Paso 9. Crear la estructura local y el inventario de recursos

**Objetivo:** crear archivos locales seguros y documentar los identificadores necesarios para la siguiente práctica.

#### Instrucciones

1. Crea la estructura local obligatoria:

   ```bash
   sudo mkdir -p /work/genai-agents-deployment
   sudo chown -R "$USER":"$USER" /work/genai-agents-deployment

   mkdir -p ~/genai-agent-labs/config
   cd /work/genai-agents-deployment
   ```

2. Crea un archivo `.gitignore` que excluya secretos, archivos locales y artefactos temporales:

   ```bash
   cat > .gitignore <<'EOF'
   .env
   .env.*
   !.env.example
   *.pem
   *.key
   *.pfx
   secrets/
   .azure/
   __pycache__/
   .pytest_cache/
   EOF
   ```

3. Crea `.env.example`. Este archivo documenta variables requeridas, pero no debe incluir valores reales de secretos.

   ```bash
   cat > .env.example <<'EOF'
   # Configuración no sensible de Azure OpenAI
   AZURE_OPENAI_ENDPOINT=
   AZURE_OPENAI_DEPLOYMENT_NAME=genai-chat-model
   AZURE_OPENAI_API_VERSION=2024-10-21

   # Se inyecta desde Azure Key Vault o Container Apps; no guardar valor local.
   AZURE_OPENAI_API_KEY=
   EOF
   ```

4. Crea un inventario de recursos sin secretos:

   ```bash
   cat > resource-inventory.md <<EOF
   # Inventario de recursos - Lab 07-00-01

   | Elemento | Valor |
   |---|---|
   | Suscripción | $(az account show --query id --output tsv) |
   | Grupo de recursos | ${RG} |
   | Región | ${LOCATION} |
   | Hub Azure AI Foundry | ${FOUNDRY_HUB} |
   | Proyecto Azure AI Foundry | ${FOUNDRY_PROJECT} |
   | Azure OpenAI | ${AOAI_NAME} |
   | Endpoint Azure OpenAI | ${AZURE_OPENAI_ENDPOINT} |
   | Modelo | ${MODEL_NAME} |
   | Versión del modelo | ${MODEL_VERSION} |
   | Deployment | ${DEPLOYMENT_NAME} |
   | API version | ${OPENAI_API_VERSION} |
   | Key Vault | ${KV_NAME} |
   | URI del secreto | ${KV_SECRET_URI} |
   | Azure Container Registry | ${ACR_NAME} |
   | Servidor ACR | ${ACR_LOGIN_SERVER} |
   | Log Analytics Workspace | ${LAW_NAME} |
   | Container Apps Environment | ${CAE_NAME} |
   | Container App | ${CA_NAME} |
   | Principal ID de identidad administrada | ${CA_PRINCIPAL_ID} |
   | Tipo de identidad | SystemAssigned |
   EOF
   ```

5. Copia los archivos no sensibles al repositorio local del curso:

   ```bash
   cp .gitignore ~/genai-agent-labs/.gitignore
   cp .env.example ~/genai-agent-labs/.env.example
   cp resource-inventory.md ~/genai-agent-labs/config/resource-inventory-lab-07-00-01.md
   ```

6. Inspecciona los archivos antes de realizar un commit:

   ```bash
   cd ~/genai-agent-labs

   git status
   git diff -- .gitignore .env.example config/resource-inventory-lab-07-00-01.md
   ```

7. Verifica que no hay valores de clave con búsquedas básicas:

   ```bash
   grep -RIn --exclude-dir=.git --exclude=".env" \
     -E "AZURE_OPENAI_API_KEY=.*[^=]$|api_key=.*[A-Za-z0-9]" \
     . || true
   ```

8. Si la revisión no muestra secretos, realiza el commit del laboratorio:

   ```bash
   git add .gitignore .env.example config/resource-inventory-lab-07-00-01.md
   git commit -m "lab-07-00-01"
   ```

#### Resultado esperado

Existe la ruta `/work/genai-agents-deployment` y el repositorio contiene archivos de configuración seguros. El archivo de inventario documenta IDs, nombres y URIs de recursos, pero no incluye claves.

#### Verificación

```bash
cd ~/genai-agent-labs

git log -1 --oneline
git status --short
cat .env.example
```

El árbol de trabajo debe estar limpio después del commit y `.env.example` no debe contener una clave real.

---

## Validación y pruebas

Realiza las siguientes validaciones al finalizar la práctica.

### Validación 1. Inventario de recursos Azure

Lista los recursos del grupo:

```bash
az resource list \
  --resource-group "$RG" \
  --query "[].{Nombre:name,Tipo:type,Ubicacion:location}" \
  --output table
```

Debes identificar como mínimo:

- Recurso Azure OpenAI.
- Azure Key Vault.
- Azure Container Registry.
- Log Analytics Workspace.
- Entorno de Azure Container Apps.
- Container App.
- Recursos asociados a Azure AI Foundry, si fueron creados por el portal.

### Validación 2. Configuración exacta del modelo

```bash
az cognitiveservices account deployment show \
  --name "$AOAI_NAME" \
  --resource-group "$RG" \
  --deployment-name "$DEPLOYMENT_NAME" \
  --query "{deployment:name,modelo:properties.model.name,version:properties.model.version,estado:properties.provisioningState}" \
  --output json
```

Criterios de aceptación:

```json
{
  "deployment": "genai-chat-model",
  "modelo": "gpt-4o-mini",
  "version": "2024-07-18",
  "estado": "Succeeded"
}
```

### Validación 3. Validación REST mediante Azure Resource Manager

Obtén un token de Azure Resource Manager:

```bash
export ARM_TOKEN="$(az account get-access-token \
  --resource https://management.azure.com/ \
  --query accessToken \
  --output tsv)"
```

Consulta los detalles de la Container App mediante REST:

```bash
curl --silent --show-error \
  --request GET \
  --header "Authorization: Bearer ${ARM_TOKEN}" \
  "https://management.azure.com${CA_RESOURCE_ID}?api-version=2024-03-01" \
  | jq '{
      nombre: .name,
      identidad: .identity.type,
      ingresoExterno: .properties.configuration.ingress.external,
      estado: .properties.provisioningState
    }'
```

Resultado esperado aproximado:

```json
{
  "nombre": "ca-genai-agent-api",
  "identidad": "SystemAssigned",
  "ingresoExterno": false,
  "estado": "Succeeded"
}
```

Elimina el token de la sesión al terminar:

```bash
unset ARM_TOKEN
```

### Validación 4. Roles de mínimo privilegio

```bash
az role assignment list \
  --assignee-object-id "$CA_PRINCIPAL_ID" \
  --all \
  --query "[].{Rol:roleDefinitionName,Scope:scope}" \
  --output table
```

Criterios de aceptación:

- `Key Vault Secrets User` tiene como ámbito el Key Vault.
- `Cognitive Services OpenAI User` tiene como ámbito el recurso Azure OpenAI.
- `AcrPull` tiene como ámbito el Azure Container Registry.
- No existen roles excesivos como `Owner` o `Contributor` asignados a la identidad administrada.

### Validación 5. Lista de comprobación final

- [ ] El grupo de recursos se llama `rg-genai-agents-eastus2`.
- [ ] La región utilizada es East US 2.
- [ ] El hub se llama `genai-agents-hub`.
- [ ] El proyecto se llama `genai-agents-project`.
- [ ] El deployment se llama `genai-chat-model`.
- [ ] El modelo es `gpt-4o-mini`, versión `2024-07-18`.
- [ ] El secreto `azure-openai-api-key` está en Key Vault.
- [ ] La Container App usa identidad `SystemAssigned`.
- [ ] La Container App tiene ingreso interno.
- [ ] Las variables no sensibles están configuradas como variables de entorno.
- [ ] La clave de Azure OpenAI se consume mediante referencia de secreto.
- [ ] El inventario local no contiene secretos.
- [ ] El repositorio Git no contiene un archivo `.env` versionado.

## Resolución de problemas

### Problema 1. El despliegue de `gpt-4o-mini` falla por cuota, modelo o versión no disponible

**Síntoma**

El comando `az cognitiveservices account deployment create` devuelve mensajes similares a:

```text
InsufficientQuota
```

o:

```text
The requested model version is not available in this region.
```

**Causa**

La suscripción no tiene cuota disponible para `gpt-4o-mini` versión `2024-07-18` en East US 2, o el modelo no está habilitado para esa suscripción y región.

**Solución**

1. Confirma que el recurso Azure OpenAI está en la región correcta:

   ```bash
   az cognitiveservices account show \
     --name "$AOAI_NAME" \
     --resource-group "$RG" \
     --query "{nombre:name,ubicacion:location,kind:kind}" \
     --output table
   ```

2. Revisa la cuota de Azure OpenAI en Azure Portal:
   - Abre **Quotas** o **Cuotas**.
   - Filtra por **Azure OpenAI**.
   - Selecciona **East US 2**.
   - Solicita incremento de cuota para el modelo requerido si procede.

3. No sustituyas silenciosamente el modelo o la versión. Solicita al instructor una región alternativa aprobada y actualiza todas las variables de la práctica de forma consistente.

4. Si ya existe un despliegue fallido, elimínalo antes de volver a crear el despliegue:

   ```bash
   az cognitiveservices account deployment delete \
     --name "$AOAI_NAME" \
     --resource-group "$RG" \
     --deployment-name "$DEPLOYMENT_NAME"
   ```

### Problema 2. La Container App no puede resolver el secreto de Key Vault

**Síntoma**

La revisión de Container Apps no inicia correctamente, los registros muestran errores de acceso a Key Vault o la referencia `keyvaultref:` aparece como no válida.

**Causa**

La identidad administrada no tiene el rol `Key Vault Secrets User`, la asignación RBAC todavía no se ha propagado, o el URI del secreto se configuró incorrectamente.

**Solución**

1. Confirma que Key Vault usa autorización RBAC:

   ```bash
   az keyvault show \
     --name "$KV_NAME" \
     --query properties.enableRbacAuthorization \
     --output tsv
   ```

   El resultado debe ser `true`.

2. Revisa la asignación de rol:

   ```bash
   az role assignment list \
     --assignee-object-id "$CA_PRINCIPAL_ID" \
     --scope "$KV_ID" \
     --query "[].roleDefinitionName" \
     --output table
   ```

3. Si el rol no existe, créalo nuevamente:

   ```bash
   az role assignment create \
     --assignee-object-id "$CA_PRINCIPAL_ID" \
     --assignee-principal-type ServicePrincipal \
     --role "Key Vault Secrets User" \
     --scope "$KV_ID"
   ```

4. Espera entre 2 y 10 minutos para la propagación RBAC.

5. Confirma que el URI apunta al secreto correcto:

   ```bash
   az keyvault secret show \
     --vault-name "$KV_NAME" \
     --name "azure-openai-api-key" \
     --query id \
     --output tsv
   ```

6. Vuelve a configurar la referencia de secreto usando el URI obtenido, sin mostrar ni copiar el valor secreto:

   ```bash
   export KV_SECRET_URI="$(az keyvault secret show \
     --vault-name "$KV_NAME" \
     --name "azure-openai-api-key" \
     --query id \
     --output tsv)"

   az containerapp secret set \
     --name "$CA_NAME" \
     --resource-group "$RG" \
     --secrets "azure-openai-api-key=keyvaultref:${KV_SECRET_URI},identityref:system"
   ```

## Limpieza

> **Advertencia:** no ejecutes esta sección si continuarás inmediatamente con la práctica `07-00-02`. Los recursos creados aquí son entradas obligatorias para la siguiente práctica.

### Limpieza recomendada al finalizar todo el lote

Para eliminar todos los recursos del laboratorio y evitar costos, elimina el grupo de recursos:

```bash
az group delete \
  --name "$RG" \
  --yes \
  --no-wait
```

Comprueba el estado de eliminación:

```bash
az group exists --name "$RG"
```

El resultado será `false` cuando la eliminación haya finalizado.

### Limpieza local opcional

Si ya no necesitas los archivos de trabajo temporales:

```bash
rm -rf /work/genai-agents-deployment
```

No elimines `~/genai-agent-labs` si contiene prácticas anteriores o pendientes.

## Resumen

En esta práctica preparaste una plataforma de despliegue GenAI en Azure con controles de seguridad básicos y operativos:

- Creaste el grupo de recursos, registraste proveedores y configuraste East US 2.
- Creaste un hub y proyecto de Azure AI Foundry.
- Desplegaste `gpt-4o-mini` versión `2024-07-18` como `genai-chat-model`.
- Guardaste la clave de Azure OpenAI en Azure Key Vault con RBAC habilitado.
- Creaste ACR, Log Analytics, un entorno de Container Apps y una aplicación base.
- Configuraste identidad administrada y roles de mínimo privilegio.
- Preparaste una ruta de autenticación moderna con Microsoft Entra ID y una ruta compatible basada en secreto de Key Vault.
- Documentaste los recursos sin exponer secretos en el repositorio.

La práctica `07-00-02` utilizará la Container App `ca-genai-agent-api`, el ACR, el endpoint de Azure OpenAI, el despliegue `genai-chat-model`, la identidad administrada y el inventario generado para construir y desplegar la API integradora.

---

# 2. Práctica 2. Proyecto Integrador Técnico

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 60 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica implementarás y desplegarás una API REST contenerizada que consume el deployment `genai-chat-model` de Azure OpenAI. La aplicación obtendrá la clave de Azure OpenAI desde Azure Key Vault mediante `DefaultAzureCredential`, sin incluir secretos en el código, la imagen OCI ni archivos versionados.

El despliegue utilizará los recursos creados en la práctica `07-00-01`: Azure Container Registry, Azure Container Apps, Azure Key Vault, Azure OpenAI, identidad administrada y Log Analytics. Al finalizar, validarás la API pública mediante HTTPS y registrarás evidencias técnicas del despliegue.

## Objetivos de aprendizaje

Al finalizar esta práctica, podrás:

- [ ] Implementar una API FastAPI con los endpoints `GET /health` y `POST /chat`.
- [ ] Obtener un secreto de Azure Key Vault mediante `DefaultAzureCredential` y `SecretClient`.
- [ ] Construir una imagen OCI reproducible sin secretos y publicarla en Azure Container Registry.
- [ ] Actualizar una Azure Container App existente con ingress externo, escalado y configuración segura.
- [ ] Validar la API desplegada, consultar registros y documentar evidencias de seguridad y operación.

## Prerrequisitos

### Conocimientos requeridos

Debes comprender los siguientes conceptos:

- FastAPI, modelos Pydantic y códigos de estado HTTP.
- Uso básico de Docker, imágenes OCI y archivos `.dockerignore`.
- Azure CLI, grupos de recursos y consultas con `--query`.
- Identidad administrada, RBAC y Azure Key Vault.
- Azure Container Apps, revisiones, ingress y escalado.
- Uso de Azure OpenAI con el SDK `openai` para Python.

### Acceso y estado previo obligatorio

Antes de comenzar, confirma que finalizaste correctamente la práctica `07-00-01` y que existen los siguientes recursos:

- Grupo de recursos `rg-genai-agents-eastus2`.
- Azure Container Registry `acrgenaiagents<sufijo>`.
- Azure Key Vault `kv-genai-agents-<sufijo>`.
- Azure Container Apps Environment creado en la práctica anterior.
- Azure Container App `ca-genai-agent-api`.
- Recurso Azure OpenAI con el deployment `genai-chat-model`.
- Secreto `azure-openai-api-key` en el Key Vault.
- Identidad administrada de `ca-genai-agent-api` con permisos para:
  - Leer secretos en Key Vault.
  - Consumir Azure OpenAI según la configuración definida en la práctica anterior.
  - Extraer imágenes desde Azure Container Registry mediante el rol `AcrPull`.

> **Importante:** No recrees recursos con nombres distintos. Esta práctica consume y actualiza el estado de infraestructura creado en `07-00-01`.

## Entorno del laboratorio

### Hardware recomendado

| Recurso | Mínimo |
|---|---:|
| Procesador | 4 núcleos |
| Memoria RAM | 16 GB |
| Espacio libre en SSD | 20 GB |
| Conectividad | Acceso HTTPS a Azure |

### Software requerido

| Herramienta | Versión requerida |
|---|---|
| Python | 3.12.8 |
| Docker Desktop o Docker Engine | 4.37.1 o 27.4.1 |
| Azure CLI | 2.67.0 o superior |
| Visual Studio Code | 1.96.4 o superior |
| FastAPI | 0.115.6 |
| Uvicorn | 0.34.0 |
| OpenAI Python SDK | 1.58.1 |
| Azure Identity | 1.19.0 |
| Azure Key Vault Secrets | 4.9.0 |

### Preparación inicial

1. Abre una terminal y verifica las versiones principales:

   ```bash
   python3 --version
   docker --version
   az version --output table
   git --version
   ```

2. Inicia sesión en Azure y selecciona la suscripción correcta:

   ```bash
   az login
   az account show --output table
   ```

3. Define las variables de trabajo. Sustituye `<sufijo>` por el sufijo usado en la práctica anterior.

   ```bash
   export RG="rg-genai-agents-eastus2"
   export SUFFIX="<sufijo>"
   export KV_NAME="kv-genai-agents-${SUFFIX}"
   export ACR_NAME="acrgenaiagents${SUFFIX}"
   export APP_NAME="ca-genai-agent-api"
   export DEPLOYMENT_NAME="genai-chat-model"
   export IMAGE_TAG="1.0.0"
   export ACR_LOGIN_SERVER="${ACR_NAME}.azurecr.io"
   export IMAGE_NAME="${ACR_LOGIN_SERVER}/genai-agent-api:${IMAGE_TAG}"
   ```

4. Verifica los recursos existentes:

   ```bash
   az group show --name "$RG" --output table

   az acr show \
     --resource-group "$RG" \
     --name "$ACR_NAME" \
     --query "{nombre:name,loginServer:loginServer,sku:sku.name}" \
     --output table

   az keyvault show \
     --resource-group "$RG" \
     --name "$KV_NAME" \
     --query "{nombre:name,uri:properties.vaultUri,rbac:properties.enableRbacAuthorization}" \
     --output table

   az containerapp show \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --query "{nombre:name,revision:properties.latestRevisionName,identidad:identity.type}" \
     --output table
   ```

**Resultado esperado**

Los comandos muestran los recursos existentes. El Key Vault debe indicar preferiblemente `rbac: true`, y la Container App debe tener una identidad administrada asignada.

**Verificación**

Consulta el deployment de Azure OpenAI disponible:

```bash
az cognitiveservices account list \
  --resource-group "$RG" \
  --query "[?kind=='OpenAI'].{nombre:name,endpoint:properties.endpoint}" \
  --output table
```

Identifica el endpoint del recurso Azure OpenAI utilizado en la práctica anterior y asígnalo a una variable:

```bash
export AZURE_OPENAI_ENDPOINT="https://<nombre-recurso-openai>.openai.azure.com/"
export KEY_VAULT_URL="https://${KV_NAME}.vault.azure.net/"
```

No almacenes la clave de Azure OpenAI en variables de entorno, archivos `.env`, historial de shell ni archivos versionados.

## Procedimiento paso a paso

### Paso 1. Preparar el directorio de trabajo y validar la identidad administrada

**Objetivo:** Preparar una ruta de código compatible con el repositorio global y confirmar que la Container App posee identidad administrada.

**Instrucciones**

1. Accede al repositorio global obligatorio:

   ```bash
   cd ~/genai-agent-labs
   git status
   ```

2. Crea el directorio de la práctica dentro del repositorio:

   ```bash
   mkdir -p work/genai-agents-deployment/app
   mkdir -p work/genai-agents-deployment/docs
   mkdir -p reports
   ```

3. Crea una ruta compatible con el alcance del laboratorio. Si tu entorno permite crear `/work`, utiliza un enlace simbólico hacia el directorio versionado:

   ```bash
   sudo mkdir -p /work
   sudo ln -sfn "$HOME/genai-agent-labs/work/genai-agents-deployment" \
     /work/genai-agents-deployment
   ```

4. Entra al proyecto:

   ```bash
   cd /work/genai-agents-deployment
   pwd
   ```

5. Consulta la identidad asignada a la Container App:

   ```bash
   export APP_PRINCIPAL_ID=$(az containerapp identity show \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --query principalId \
     --output tsv)

   echo "Principal ID de la aplicación: ${APP_PRINCIPAL_ID}"
   ```

6. Consulta las asignaciones RBAC existentes de esa identidad:

   ```bash
   az role assignment list \
     --assignee "$APP_PRINCIPAL_ID" \
     --all \
     --query "[].{rol:roleDefinitionName,scope:scope}" \
     --output table
   ```

**Resultado esperado**

La ruta actual debe ser `/work/genai-agents-deployment`, pero los archivos realmente estarán bajo `~/genai-agent-labs/work/genai-agents-deployment`, por lo que podrán incluirse en el repositorio Git.

La Container App debe mostrar un `principalId` y roles relacionados con:

- Lectura de secretos de Key Vault, como `Key Vault Secrets User`.
- Extracción de imágenes de ACR, como `AcrPull`.

**Verificación**

Confirma que el secreto existe sin mostrar su valor:

```bash
az keyvault secret show \
  --vault-name "$KV_NAME" \
  --name "azure-openai-api-key" \
  --query "{nombre:name,habilitado:attributes.enabled,actualizado:attributes.updated}" \
  --output table
```

> No uses `--query value`, `--output tsv` ni comandos que impriman el valor del secreto.

---

### Paso 2. Crear la API FastAPI segura

**Objetivo:** Implementar los endpoints de salud y conversación utilizando Azure OpenAI, Azure Key Vault e identidad administrada.

**Instrucciones**

1. Crea el archivo `app/main.py`:

   ```bash
   cat > app/main.py <<'PY'
   import logging
   import os
   from functools import lru_cache

   from azure.identity import DefaultAzureCredential
   from azure.keyvault.secrets import SecretClient
   from fastapi import FastAPI, HTTPException
   from openai import APIError, APITimeoutError, AzureOpenAI
   from pydantic import BaseModel, Field, field_validator

   logging.basicConfig(
       level=os.getenv("LOG_LEVEL", "INFO"),
       format="%(asctime)s %(levelname)s %(name)s %(message)s",
   )
   logger = logging.getLogger("genai_agent_api")

   SYSTEM_MESSAGE = (
       "Eres un asistente técnico empresarial. Responde en español, "
       "de forma clara, precisa y sin inventar información."
   )

   AZURE_OPENAI_ENDPOINT = os.getenv("AZURE_OPENAI_ENDPOINT", "").strip()
   AZURE_OPENAI_DEPLOYMENT = os.getenv("AZURE_OPENAI_DEPLOYMENT", "").strip()
   KEY_VAULT_URL = os.getenv("KEY_VAULT_URL", "").strip()
   KEY_VAULT_SECRET_NAME = os.getenv(
       "KEY_VAULT_SECRET_NAME", "azure-openai-api-key"
   ).strip()

   app = FastAPI(
       title="API de Agente GenAI",
       version="1.0.0",
       description="API segura para consumir Azure OpenAI mediante Key Vault.",
   )


   class ChatRequest(BaseModel):
       message: str = Field(
           ...,
           min_length=1,
           max_length=2000,
           description="Mensaje del usuario con un máximo de 2000 caracteres.",
       )

       @field_validator("message")
       @classmethod
       def validate_message(cls, value: str) -> str:
           normalized = value.strip()
           if not normalized:
               raise ValueError("El mensaje no puede estar vacío.")
           return normalized


   @lru_cache(maxsize=1)
   def get_openai_client() -> AzureOpenAI:
       if not AZURE_OPENAI_ENDPOINT or not AZURE_OPENAI_DEPLOYMENT:
           logger.error("Falta configuración no sensible de Azure OpenAI.")
           raise RuntimeError("Configuración de Azure OpenAI incompleta.")

       if not KEY_VAULT_URL:
           logger.error("Falta la URL de Azure Key Vault.")
           raise RuntimeError("Configuración de Key Vault incompleta.")

       try:
           credential = DefaultAzureCredential(
               exclude_interactive_browser_credential=True
           )
           secret_client = SecretClient(
               vault_url=KEY_VAULT_URL,
               credential=credential,
           )
           api_key = secret_client.get_secret(KEY_VAULT_SECRET_NAME).value

           if not api_key:
               raise RuntimeError("El secreto requerido no contiene un valor.")

           return AzureOpenAI(
               azure_endpoint=AZURE_OPENAI_ENDPOINT,
               api_key=api_key,
               api_version="2024-10-21",
               timeout=30.0,
               max_retries=1,
           )
       except Exception as error:
           logger.error(
               "No fue posible inicializar el cliente de Azure OpenAI. tipo_error=%s",
               type(error).__name__,
           )
           raise RuntimeError(
               "No fue posible obtener la configuración segura del servicio."
           ) from error


   @app.get("/health")
   def health() -> dict:
       return {
           "status": "available",
           "service": "genai-agent-api",
           "version": "1.0.0",
       }


   @app.post("/chat")
   def chat(request: ChatRequest) -> dict:
       try:
           client = get_openai_client()
           response = client.chat.completions.create(
               model=AZURE_OPENAI_DEPLOYMENT,
               messages=[
                   {"role": "system", "content": SYSTEM_MESSAGE},
                   {"role": "user", "content": request.message},
               ],
               max_tokens=500,
               temperature=0.2,
           )

           answer = response.choices[0].message.content
           if not answer:
               raise RuntimeError("El modelo no devolvió contenido.")

           return {
               "response": answer,
               "model": AZURE_OPENAI_DEPLOYMENT,
           }

       except (APITimeoutError, APIError) as error:
           logger.error(
               "Error controlado al invocar Azure OpenAI. tipo_error=%s",
               type(error).__name__,
           )
           raise HTTPException(
               status_code=502,
               detail="No fue posible obtener una respuesta del modelo.",
           ) from error

       except RuntimeError as error:
           logger.error(
               "Error de configuración o dependencia. tipo_error=%s",
               type(error).__name__,
           )
           raise HTTPException(
               status_code=503,
               detail="El servicio no está listo para procesar solicitudes.",
           ) from error

       except Exception as error:
           logger.error(
               "Error no controlado durante la solicitud. tipo_error=%s",
               type(error).__name__,
           )
           raise HTTPException(
               status_code=500,
               detail="Error interno al procesar la solicitud.",
           ) from error
   PY
   ```

2. Revisa que el código no contenga una clave ni un valor secreto:

   ```bash
   grep -n "azure-openai-api-key" app/main.py
   grep -n "DefaultAzureCredential" app/main.py
   grep -n "SecretClient" app/main.py
   ```

3. Observa las decisiones de seguridad incorporadas:

   - El nombre del secreto puede aparecer en el código porque no es secreto.
   - La clave se obtiene en tiempo de ejecución desde Key Vault.
   - `DefaultAzureCredential` utiliza identidad administrada en Azure.
   - La entrada está limitada a 2.000 caracteres.
   - Los errores HTTP no incluyen secretos, prompts ni trazas internas.
   - Los registros sólo guardan tipos de error, no contenido de solicitudes o respuestas.
   - La conversación incorpora un mensaje de sistema fijo.

**Resultado esperado**

El archivo `app/main.py` contiene una API con:

- `GET /health`.
- `POST /chat`.
- Validación de entrada mediante Pydantic.
- Recuperación del secreto `azure-openai-api-key`.
- Llamada al deployment `genai-chat-model`.
- Uso de la versión de API `2024-10-21`.

**Verificación**

Valida la sintaxis Python:

```bash
python3 -m py_compile app/main.py
```

El comando no debe mostrar errores.

---

### Paso 3. Crear dependencias, imagen OCI y exclusiones de Docker

**Objetivo:** Crear una definición de contenedor reproducible que no copie secretos ni artefactos locales a la imagen.

**Instrucciones**

1. Crea `requirements.txt` con versiones fijas:

   ```bash
   cat > requirements.txt <<'EOF'
   fastapi==0.115.6
   uvicorn[standard]==0.34.0
   openai==1.58.1
   azure-identity==1.19.0
   azure-keyvault-secrets==4.9.0
   pydantic==2.10.4
   EOF
   ```

2. Crea el archivo `.dockerignore`:

   ```bash
   cat > .dockerignore <<'EOF'
   .git
   .gitignore
   .venv
   __pycache__
   *.py[cod]
   *.log
   .env
   .env.*
   !.env.example
   .pytest_cache
   .mypy_cache
   reports
   tests
   docs
   data
   config
   prompts
   README.md
   EOF
   ```

3. Crea el `Dockerfile`:

   ```bash
   cat > Dockerfile <<'EOF'
   FROM python:3.12.8-slim

   ENV PYTHONDONTWRITEBYTECODE=1 \
       PYTHONUNBUFFERED=1 \
       PORT=8000

   WORKDIR /app

   COPY requirements.txt .

   RUN pip install --no-cache-dir --upgrade pip \
       && pip install --no-cache-dir -r requirements.txt \
       && useradd --create-home --uid 10001 appuser

   COPY app ./app

   RUN chown -R appuser:appuser /app

   USER appuser

   EXPOSE 8000

   CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
   EOF
   ```

4. Construye la imagen local:

   ```bash
   docker build \
     --tag "$IMAGE_NAME" \
     --file Dockerfile \
     .
   ```

5. Inspecciona la imagen creada:

   ```bash
   docker image inspect "$IMAGE_NAME" \
     --format '{{.RepoTags}} {{.Config.User}} {{.Config.ExposedPorts}}'
   ```

**Resultado esperado**

La imagen se construye correctamente y utiliza:

- Imagen base `python:3.12.8-slim`.
- Usuario no privilegiado `appuser`.
- Puerto `8000`.
- Dependencias con versiones fijadas.
- Exclusiones de `.env`, `.git`, entornos virtuales, reportes y cachés.

**Verificación**

Comprueba el historial de capas sin exponer secretos:

```bash
docker history --no-trunc "$IMAGE_NAME"
```

No debe aparecer una clave de Azure OpenAI, una variable `AZURE_OPENAI_API_KEY` ni comandos que recuperen o impriman secretos.

---

### Paso 4. Probar localmente el endpoint de salud

**Objetivo:** Verificar que la imagen OCI se ejecuta localmente en el puerto obligatorio `8000`.

**Instrucciones**

1. Asegúrate de que ningún servicio usa el puerto `8000`:

   ```bash
   ss -ltnp | grep ':8000' || true
   ```

2. Ejecuta el contenedor localmente sin secretos:

   ```bash
   docker run --rm \
     --name genai-agent-api-local \
     --publish 8000:8000 \
     "$IMAGE_NAME"
   ```

3. En otra terminal, prueba el endpoint de salud:

   ```bash
   curl --silent --show-error \
     http://127.0.0.1:8000/health | jq
   ```

4. Detén el contenedor con `Ctrl+C` en la primera terminal.

**Resultado esperado**

La respuesta debe ser similar a la siguiente:

```json
{
  "status": "available",
  "service": "genai-agent-api",
  "version": "1.0.0"
}
```

**Verificación**

No es obligatorio probar localmente `POST /chat`, porque el contenedor local no posee una identidad administrada de Azure. La prueba funcional completa de `/chat` se realizará tras desplegar la imagen en Azure Container Apps, donde `DefaultAzureCredential` utilizará la identidad administrada asignada.

---

### Paso 5. Publicar la imagen en Azure Container Registry

**Objetivo:** Autenticar Docker contra ACR y publicar la imagen etiquetada como versión `1.0.0`.

**Instrucciones**

1. Autentica Docker contra Azure Container Registry:

   ```bash
   az acr login --name "$ACR_NAME"
   ```

2. Publica la imagen:

   ```bash
   docker push "$IMAGE_NAME"
   ```

3. Lista los repositorios e imágenes disponibles en ACR:

   ```bash
   az acr repository show-tags \
     --name "$ACR_NAME" \
     --repository "genai-agent-api" \
     --output table
   ```

4. Consulta el manifiesto de la etiqueta publicada:

   ```bash
   az acr repository show \
     --name "$ACR_NAME" \
     --image "genai-agent-api:${IMAGE_TAG}" \
     --query "{digest:digest,createdTime:createdTime,architecture:architecture}" \
     --output table
   ```

**Resultado esperado**

El repositorio `genai-agent-api` debe contener la etiqueta `1.0.0`.

**Verificación**

La referencia completa de imagen debe coincidir con este patrón:

```text
acrgenaiagents<sufijo>.azurecr.io/genai-agent-api:1.0.0
```

---

### Paso 6. Actualizar Azure Container Apps con configuración segura

**Objetivo:** Actualizar la Container App existente con la nueva imagen, ingress HTTPS, escalado y variables no sensibles.

**Instrucciones**

1. Configura el acceso de la identidad administrada al registro. Este comando no crea un recurso nuevo; configura la aplicación existente para usar su identidad del sistema al extraer la imagen:

   ```bash
   az containerapp registry set \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --server "$ACR_LOGIN_SERVER" \
     --identity system
   ```

2. Habilita o confirma el ingress externo hacia el puerto `8000`:

   ```bash
   az containerapp ingress enable \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --type external \
     --target-port 8000 \
     --transport auto
   ```

3. Actualiza la Container App con la imagen y únicamente variables no sensibles:

   ```bash
   az containerapp update \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --image "$IMAGE_NAME" \
     --min-replicas 1 \
     --max-replicas 2 \
     --set-env-vars \
       PORT=8000 \
       AZURE_OPENAI_ENDPOINT="$AZURE_OPENAI_ENDPOINT" \
       AZURE_OPENAI_DEPLOYMENT="$DEPLOYMENT_NAME" \
       KEY_VAULT_URL="$KEY_VAULT_URL" \
       KEY_VAULT_SECRET_NAME="azure-openai-api-key" \
       LOG_LEVEL=INFO
   ```

4. Espera a que se cree la nueva revisión y consulta su estado:

   ```bash
   az containerapp show \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --query "{
       fqdn:properties.configuration.ingress.fqdn,
       revision:properties.latestRevisionName,
       imagen:properties.template.containers[0].image,
       minReplicas:properties.template.scale.minReplicas,
       maxReplicas:properties.template.scale.maxReplicas
     }" \
     --output json
   ```

5. Obtén la FQDN pública:

   ```bash
   export FQDN=$(az containerapp show \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --query properties.configuration.ingress.fqdn \
     --output tsv)

   export PUBLIC_URL="https://${FQDN}"

   echo "$PUBLIC_URL"
   ```

**Resultado esperado**

La Container App debe ejecutar la imagen `genai-agent-api:1.0.0`, exponer ingress externo HTTPS y mantener entre una y dos réplicas.

**Verificación**

Consulta la configuración visible de variables de entorno:

```bash
az containerapp show \
  --resource-group "$RG" \
  --name "$APP_NAME" \
  --query "properties.template.containers[0].env" \
  --output table
```

Debe aparecer la configuración no sensible:

- `PORT`
- `AZURE_OPENAI_ENDPOINT`
- `AZURE_OPENAI_DEPLOYMENT`
- `KEY_VAULT_URL`
- `KEY_VAULT_SECRET_NAME`
- `LOG_LEVEL`

No debe aparecer una variable llamada `AZURE_OPENAI_API_KEY` ni el valor de la clave.

---

### Paso 7. Validar la API pública y revisar los registros

**Objetivo:** Probar los endpoints HTTPS, validar la integración con Azure OpenAI y revisar registros operativos.

**Instrucciones**

1. Prueba el endpoint de salud:

   ```bash
   curl --silent --show-error \
     --write-out "\nHTTP %{http_code}\n" \
     "${PUBLIC_URL}/health"
   ```

2. Prueba una solicitud válida a `POST /chat`:

   ```bash
   curl --silent --show-error \
     --request POST \
     --header "Content-Type: application/json" \
     --data '{"message":"Explica brevemente qué es una identidad administrada en Azure."}' \
     --write-out "\nHTTP %{http_code}\n" \
     "${PUBLIC_URL}/chat" | jq
   ```

3. Prueba una entrada inválida para validar el límite básico de entrada:

   ```bash
   curl --silent --show-error \
     --request POST \
     --header "Content-Type: application/json" \
     --data '{"message":"   "}' \
     --write-out "\nHTTP %{http_code}\n" \
     "${PUBLIC_URL}/chat" | jq
   ```

4. Consulta los registros recientes de la Container App:

   ```bash
   az containerapp logs show \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --type console \
     --tail 50
   ```

5. Obtén el identificador del workspace de Log Analytics asociado al entorno de Container Apps:

   ```bash
   export ENVIRONMENT_ID=$(az containerapp show \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --query properties.managedEnvironmentId \
     --output tsv)

   echo "$ENVIRONMENT_ID"
   ```

6. En Azure Portal, abre el recurso de Log Analytics asociado al Container Apps Environment y ejecuta una consulta similar a la siguiente:

   ```kusto
   ContainerAppConsoleLogs_CL
   | where ContainerAppName_s == "ca-genai-agent-api"
   | project TimeGenerated, Log_s, RevisionName_s
   | order by TimeGenerated desc
   | take 50
   ```

   Si el workspace utiliza tablas nuevas, consulta primero las tablas disponibles o utiliza:

   ```kusto
   search "ca-genai-agent-api"
   | order by TimeGenerated desc
   | take 50
   ```

**Resultado esperado**

- `GET /health` devuelve HTTP `200`.
- `POST /chat` devuelve HTTP `200` y un objeto JSON con los campos `response` y `model`.
- La entrada vacía devuelve un error de validación HTTP `422`.
- Los registros muestran inicio de la aplicación, solicitudes HTTP o errores controlados, sin claves, prompts completos ni respuestas completas.

**Verificación**

Ejemplo de respuesta válida esperada:

```json
{
  "response": "Una identidad administrada es una identidad administrada por Azure que permite a una aplicación autenticarse ante servicios compatibles sin almacenar credenciales en el código.",
  "model": "genai-chat-model"
}
```

El texto exacto puede variar. La verificación principal es que el código HTTP sea `200`, que el campo `model` sea `genai-chat-model` y que no se exponga ningún secreto.

## Validación y pruebas

Completa la siguiente lista de validación antes de finalizar:

| Validación | Comando o evidencia | Resultado esperado |
|---|---|---|
| Código Python válido | `python3 -m py_compile app/main.py` | Sin errores |
| Imagen construida | `docker image ls \| grep genai-agent-api` | Etiqueta `1.0.0` |
| Endpoint local de salud | `curl http://127.0.0.1:8000/health` | HTTP 200 |
| Imagen publicada | `az acr repository show-tags ...` | Etiqueta `1.0.0` |
| App actualizada | `az containerapp show ...` | Imagen ACR correcta |
| Ingress externo HTTPS | `echo "$PUBLIC_URL"` | URL HTTPS pública |
| Salud remota | `curl "${PUBLIC_URL}/health"` | HTTP 200 |
| Chat remoto | `curl -X POST "${PUBLIC_URL}/chat"` | HTTP 200 |
| Escalado | Consulta de `minReplicas` y `maxReplicas` | 1 y 2 |
| Logs revisados | `az containerapp logs show ...` | Sin secretos |
| Configuración visible | Consulta de variables de Container Apps | Sin clave API |

Crea un documento de arquitectura y evidencias sin incluir datos sensibles:

```bash
cat > docs/07-00-02-arquitectura.md <<EOF
# Arquitectura final del despliegue

## Componentes

Cliente HTTPS
  -> Azure Container Apps: ${APP_NAME}
  -> API FastAPI: GET /health y POST /chat
  -> Azure Key Vault: ${KV_NAME}
  -> Azure OpenAI deployment: ${DEPLOYMENT_NAME}
  -> Azure Container Registry: ${ACR_NAME}
  -> Azure Monitor y Log Analytics

## Decisiones de seguridad

1. La clave de Azure OpenAI no se almacena en código, Dockerfile, imagen ni variables visibles.
2. La API recupera el secreto azure-openai-api-key desde Azure Key Vault.
3. DefaultAzureCredential utiliza la identidad administrada de la Container App en Azure.
4. Sólo se configuran variables no sensibles en Azure Container Apps.
5. Los logs no registran prompts, respuestas ni valores de secretos.
6. La API limita los mensajes a 2000 caracteres y devuelve errores HTTP genéricos.

## URL pública

${PUBLIC_URL}

## Evidencias de prueba

- GET /health: HTTP 200.
- POST /chat: HTTP 200 con model=${DEPLOYMENT_NAME}.
- Entrada vacía: HTTP 422.
- Imagen publicada: ${IMAGE_NAME}.
- Escalado configurado: mínimo 1, máximo 2 réplicas.
- Registros revisados en Azure Container Apps y Log Analytics.
EOF
```

Crea además un reporte operativo mínimo:

```bash
cat > ~/genai-agent-labs/reports/07-00-02-evidencias.json <<EOF
{
  "lab_id": "07-00-02",
  "container_app": "${APP_NAME}",
  "public_url": "${PUBLIC_URL}",
  "image": "${IMAGE_NAME}",
  "deployment": "${DEPLOYMENT_NAME}",
  "health_expected_status": 200,
  "chat_expected_status": 200,
  "min_replicas": 1,
  "max_replicas": 2,
  "secret_exposed": false
}
EOF
```

Finalmente, revisa los cambios y crea un commit:

```bash
cd ~/genai-agent-labs

git status
git add work/genai-agents-deployment reports/07-00-02-evidencias.json
git commit -m "lab-07-00-02"
```

> No agregues archivos `.env`, claves, credenciales de Azure CLI, resultados que contengan secretos ni volcados completos de registros.

## Solución de problemas

### Problema 1: `POST /chat` devuelve HTTP 503 y los registros indican error de autenticación o acceso a Key Vault

**Síntomas**

- `GET /health` devuelve HTTP 200.
- `POST /chat` devuelve HTTP 503.
- Los logs muestran mensajes como `ClientAuthenticationError`, `ResourceNotFoundError` o errores al inicializar el cliente.
- La aplicación no puede recuperar el secreto `azure-openai-api-key`.

**Causa**

La identidad administrada de `ca-genai-agent-api` no tiene el rol de plano de datos necesario sobre el Key Vault, el nombre del secreto es incorrecto o la URL configurada en `KEY_VAULT_URL` no coincide con el vault existente.

**Corrección**

1. Confirma el nombre y estado del secreto:

   ```bash
   az keyvault secret show \
     --vault-name "$KV_NAME" \
     --name "azure-openai-api-key" \
     --query "{nombre:name,habilitado:attributes.enabled}" \
     --output table
   ```

2. Confirma el `principalId` de la Container App:

   ```bash
   az containerapp identity show \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --output json
   ```

3. Solicita al administrador o usa permisos `User Access Administrator` para asignar el rol mínimo requerido a la identidad administrada:

   ```bash
   export KV_ID=$(az keyvault show \
     --name "$KV_NAME" \
     --resource-group "$RG" \
     --query id \
     --output tsv)

   az role assignment create \
     --assignee-object-id "$APP_PRINCIPAL_ID" \
     --assignee-principal-type ServicePrincipal \
     --role "Key Vault Secrets User" \
     --scope "$KV_ID"
   ```

4. Verifica que `KEY_VAULT_URL` tenga este formato:

   ```text
   https://kv-genai-agents-<sufijo>.vault.azure.net/
   ```

5. Reinicia mediante una nueva revisión o ejecuta nuevamente `az containerapp update`.

### Problema 2: La Container App no inicia o muestra errores al descargar la imagen desde ACR

**Síntomas**

- La nueva revisión no queda saludable.
- Azure Container Apps muestra errores de `ImagePullFailure`, `Unauthorized` o `denied`.
- La FQDN responde con error de disponibilidad.
- Los logs del sistema indican que la imagen no se pudo descargar.

**Causa**

La identidad administrada de la Container App no tiene el rol `AcrPull` sobre Azure Container Registry, el servidor de registro está mal configurado o la etiqueta `1.0.0` no fue publicada correctamente.

**Corrección**

1. Verifica que la imagen exista:

   ```bash
   az acr repository show-tags \
     --name "$ACR_NAME" \
     --repository "genai-agent-api" \
     --output table
   ```

2. Consulta el identificador del registro:

   ```bash
   export ACR_ID=$(az acr show \
     --resource-group "$RG" \
     --name "$ACR_NAME" \
     --query id \
     --output tsv)
   ```

3. Asigna el rol `AcrPull` a la identidad administrada:

   ```bash
   az role assignment create \
     --assignee-object-id "$APP_PRINCIPAL_ID" \
     --assignee-principal-type ServicePrincipal \
     --role "AcrPull" \
     --scope "$ACR_ID"
   ```

4. Configura nuevamente el registro para usar la identidad del sistema:

   ```bash
   az containerapp registry set \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --server "$ACR_LOGIN_SERVER" \
     --identity system
   ```

5. Ejecuta de nuevo la actualización con la imagen correcta:

   ```bash
   az containerapp update \
     --resource-group "$RG" \
     --name "$APP_NAME" \
     --image "$IMAGE_NAME"
   ```

## Limpieza

Esta práctica usa recursos compartidos creados en `07-00-01`, por lo que **no debes eliminar** el grupo de recursos, Key Vault, ACR, Azure OpenAI, Container Apps Environment ni la Container App desplegada.

Realiza únicamente la limpieza local opcional:

```bash
docker rm -f genai-agent-api-local 2>/dev/null || true
docker image rm "$IMAGE_NAME" 2>/dev/null || true

unset AZURE_OPENAI_ENDPOINT
unset KEY_VAULT_URL
unset FQDN
unset PUBLIC_URL
```

Conserva los siguientes artefactos en el repositorio:

- `work/genai-agents-deployment/app/main.py`
- `work/genai-agents-deployment/Dockerfile`
- `work/genai-agents-deployment/.dockerignore`
- `work/genai-agents-deployment/requirements.txt`
- `work/genai-agents-deployment/docs/07-00-02-arquitectura.md`
- `reports/07-00-02-evidencias.json`

## Resumen

En esta práctica construiste una API FastAPI para consumir Azure OpenAI mediante el deployment `genai-chat-model`. La aplicación usa `DefaultAzureCredential` y `SecretClient` para recuperar la clave desde Azure Key Vault sin exponerla en código, configuración visible o imagen de contenedor.

También construiste y publicaste una imagen OCI versionada en Azure Container Registry, actualizaste una Azure Container App existente con ingress HTTPS, escalado de una a dos réplicas y variables no sensibles. Finalmente, validaste los endpoints públicos, revisaste registros operativos y documentaste la arquitectura, los controles de seguridad y las evidencias de despliegue.
