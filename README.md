**Roteiro para Implementação do IAP**

**Instruções**

Este roteiro descreve como implementar o IAP (Identity-Aware Proxy) em um serviço de backend do Cloud Run. O IAP fornece uma camada de segurança adicional para seus aplicativos, exigindo que os usuários se autentiquem antes de poderem acessar.

**Requisitos**

* Um projeto do Cloud Platform com a API de serviço IAP habilitada.
* Um serviço de backend do Cloud Run.
* Um domínio DNS para o serviço de backend.

**Observação**
Roteiro definido na implementação do IAP no ambiente de desenvolvimento. Portanto os nomes de aplicações e serviços utilizados como exemplo no roteiro, seguem o padrão do ambiente dev.

**Passos**

1. **Definir as variáveis de ambiente.**

As seguintes variáveis de ambiente são usadas neste roteiro:

* `PROJECT_ID`: O ID do projeto do Cloud Platform.
* `PROJECT_NUMBER`: O número do projeto do Cloud Platform.
* `REGION`: A região do Cloud Platform onde o serviço de backend está localizado.
* `APP_NAME`: O nome do serviço de backend.
* `DOMAIN`: O domínio DNS para o serviço de backend.

```
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
export REGION=us-central1
export APP_NAME=dev-sgmra
export DOMAIN=sgmra.dev.greendatabr.com
```

2. **Habilitar as APIs necessárias.**

```
gcloud services enable iap.googleapis.com cloudresourcemanager.googleapis.com cloudidentity.googleapis.com compute.googleapis.com
```

3. **Criar um NEG (grupo de endpoints de rede) para o serviço.**

```
gcloud compute network-endpoint-groups create $APP_NAME-ui-iap-neg --project $PROJECT_ID --region $REGION --network-endpoint-type=serverless --cloud-run-service=$APP_NAME
```

4. **Criar um serviço de backend.**

```
gcloud compute backend-services create $APP_NAME-ui-iap-backend --global
```

5. **Adicionar o serverless NEG como backend do serviço de backend criado.**

```
gcloud compute backend-services add-backend $APP_NAME-ui-iap-backend --global --network-endpoint-group=$APP_NAME-ui-iap-neg --network-endpoint-group-region=$REGION
```

6. **Criar um url map para rotear as solicitações de entrada para o serviço de backend.**

```
gcloud compute url-maps create $APP_NAME-ui-iap-url-map --default-service $APP_NAME-ui-iap-backend
```

7. **Reservar um endereço de ip statico ipv4 para o load balancer.**

```
gcloud compute addresses create $APP_NAME-ui-iap-ip --network-tier=PREMIUM --ip-version=IPV4 --global
```

8. **Obter o endereço de ip gerado no item anterior.**

```
gcloud compute addresses list --filter dev-sgmra-ui-iap-ip --format='value(ADDRESS)'
```

***Este ip retornado deve ser utilizado para registro so subdominio no provedor utilizado (e.g. GoDaddy, Hostinher, KingHost). Importante que esse registro de DNS seja feito antes do próximo passo.***

9. **Criar um certificado SSL gerenciado pelo Google.**

```
gcloud compute ssl-certificates create $APP_NAME-ui-iap-cert \
    --description=$APP_NAME-ui-iap-cert \
    --domains=$DOMAIN \
    --global
```

10. **Criar um proxy HTTPS de destino para rotear solicitações para o url map.**

```
gcloud compute target-https-proxies create $APP_NAME-ui-iap-http-proxy \
    --ssl-certificates $APP_NAME-ui-iap-cert \
    --url-map $APP_NAME-ui-iap-url-map
```

11. **Criar uma regra de encaminhamento para encaminhar as solicitações recebidas para proxy https.**

```
gcloud compute forwarding-rules create $APP_NAME-ui-iap-forwarding-rule \
    --load-balancing-scheme=EXTERNAL \
    --network-tier=PREMIUM \
    --address=$APP_NAME-ui-iap-ip \
    --global \
    --ports=443 \
    --target-https-proxy $APP_NAME-ui-iap-http-proxy
```

12. **Criar a tela de consentimento OAuth.**

```
export USER_EMAIL=$(gcloud config list account --format "value(core.account)")

gcloud alpha iap oauth-brands create \
    --application_title=$APP_NAME \
    --support_email=$USER_EMAIL
```

13. **Criar um cliente usando a tela de consentimento da etapa anterior.**

```
gcloud alpha iap oauth-clients create \
        projects/$PROJECT_ID/brands/$PROJECT_NUMBER \
        --display_name=$APP_NAME
```

14. **Armazenar nas variaveis de ambiente o nome do cliente, ID e secret.**

```
export CLIENT_NAME=$(gcloud alpha iap oauth-clients list \
        projects/$PROJECT_NUMBER/brands/$PROJECT_NUMBER --format='value(name)' \
        --filter="displayName:$APP_NAME")

export CLIENT_ID=${CLIENT_NAME##*/}

export CLIENT_SECRET=$(gcloud alpha iap oauth-clients describe $CLIENT_NAME --format='value(secret)')
```

15. **No cloud console, no projeto correto, ir até o menu sanduiche (esquerda superior) e navegar até APIs & Services > OAuth consent screen. Em seguida clicar em "MAKE EXTERNAL" e selecionar "Testing" (ou "In production") e confirmar.**

16. **Crir uma conta de serviço para o IAP.**

```	
gcloud beta services identity create --service=iap.googleapis.com --project=$PROJECT_ID

export IAP_SERVICE_ACCOUNT=service-$PROJECT_NUMBER@gcp-sa-iap.iam.gserviceaccount.com
```

17. **Conceder a conta de serviço do IAP criada na etapa anterior a permissão de chamar o serviço do Cloud Run.**

```
gcloud run services add-iam-policy-binding $APP_NAME \
    --member=serviceAccount:$IAP_SERVICE_ACCOUNT  \
    --role='roles/run.invoker'
```

18. **Ativar o IAP com escopo global ou regional . Use o ID e a chave secreta do cliente OAuth da etapa anterior (global ou regional? dependendo do serviço de backend do balanceador criado no item 4 - criamos com escopo global).**
	
***Escopo global (utilizar este, baseado no serviço de backend atual):***
```
gcloud compute backend-services update $APP_NAME-ui-iap-backend --global --iap=enabled
```

***Escopo Regional:***
```
gcloud compute backend-services update $APP_NAME-ui-iap-backend --region $REGION --iap=enabled
```

19. **Ativar o IAP no serviço de backend.**

```
gcloud iap web enable --resource-type=backend-services \
        --oauth2-client-id=$CLIENT_ID \
        --oauth2-client-secret=$CLIENT_SECRET \
        --service=$APP_NAME-ui-iap-backend
```

20. **Verificar se o certificado SSL criado esta ativo.**

```
gcloud compute ssl-certificates list --format='value(MANAGED_STATUS)'
```

***Se ativo, seguir. Caso contrario aguardar e repetir este passo até obter status ativo. Pode demorar.***

21. **Obter a url do serviço.**

```
echo https://$DOMAIN
```

***Clicar no link retornado para abrir a pagina da aplicação e testar.***
***Provavelmente será negado o acesso.***

22. **Adicionar uma politica IAM ao usuario ou a um grupo de usuarios.**

```
gcloud iap web add-iam-policy-binding \
        --resource-type=backend-services \
        --service=$APP_NAME-ui-iap-backend \
        --member=group:$APP_NAME@greendatabr.com \
        --role='roles/iap.httpsResourceAccessor'
```

23. **Repetir o passo 20. Provavelmente terá acesso a aplicação que esta no Cloud Run.**