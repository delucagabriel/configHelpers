# Configurando o Insomnia para chamadas ao Sharepoint REST

## Registrando o app no seu tenant

> Referências:
>
> - [Accessing SharePoint Data using Postman (SharePoint REST API)](https://medium.com/@anoopt/accessing-sharepoint-data-using-postman-sharepoint-rest-api-76b70630bcbf)
> - [In 4 steps access SharePoint online data using postman tool](https://global-sharepoint.com/sharepoint-online/in-4-steps-access-sharepoint-online-data-using-postman-tool/)

Vá até o caminho: <https://{YourTenantName}.sharepoint.com/_layouts/15/appregnew.aspx>
![AppRegNew](/sharepointConfig/imgs/1_VqLBfZ-QNko8FbGxir9wbA.png)

Preencha os detalhes dessa página conforme a tabela a seguir e clique em “Criar”.

Field | Value
------|------
Client Id | Clique em Generate
Client Secret | Clique em Generate
Title | Nome do seu aplicativo
App Domain | localhost
Redirect URI | "https://localhost"

Copie o ID do cliente gerado e o segredo do cliente no bloco de notas (ou em qualquer um de seu editor favorito), pois precisaremos deles posteriormente.
Agora que o aplicativo está registrado, precisamos fornecer ao aplicativo algumas permissões para que ele possa acessar os dados. Para fazer isso, navegue até a página “appinv.aspx” (com a qual você pode conceder permissões a um aplicativo). O URL dessa página será semelhante ao abaixo

> <https://{YourTenantName}.sharepoint.com/_layouts/15/appinv.aspx>

Nessa página, cole o ID do cliente na caixa de texto “App Id” e clique em “Lookup”. Isso irá carregar os detalhes do aplicativo que registramos anteriormente

![appInv](/sharepointConfig/imgs/1_nhI4ttkgDmZpcywxUaV8Cw.png)

No “XML de solicitação de permissão” cole o seguinte XML. Este XML diz que o aplicativo pode ter controle total sobre a web atual (que é tudo de que preciso neste caso). Se você precisar conceder permissões diferentes, dê uma olhada neste artigo da Microsoft.

```xml
<AppPermissionRequests AllowAppOnlyPolicy="true">
  <AppPermissionRequest Scope="http://sharepoint/content/sitecollection/web" Right="FullControl"/>
</AppPermissionRequests>
```

Uma vez adicionado, clique em “Criar”. Na próxima tela, clique em “Trust It” e isso significará que o aplicativo terá as permissões necessárias.

![Token trust](/sharepointConfig/imgs/1_g01kA4K3v58tjLI7Aau00w.png)

Isso conclui os bits relacionados ao SharePoint. Agora, vamos para o Insomnia.

## Obtendo o ID realm

Abra o insomnia e crie uma nova requisição GET para o caminho <https://{YourTenantName}.sharepoint.com/_vti_bin/client.svc/>

Em Header, insira:
Header | Value
---|---
Authorization | Bearer

![newRequest](/sharepointConfig/imgs/Captura%20de%20tela%20de%202020-08-08%2017-53-23.png)
![requisição GET](/sharepointConfig/imgs/Captura%20de%20tela%20de%202020-08-08%2017-59-48.png)

Clique em send e receberá a resposta:

```xml
<?xml version="1.0" encoding="utf-8"?>
<m:error
  xmlns:m="http://schemas.microsoft.com/ado/2007/08/dataservices/metadata">
  <m:code>-2147024891, System.UnauthorizedAccessException</m:code>
  <m:message xml:lang="en-US">Access denied. You do not have permission to perform this action or access this resource.</m:message>
</m:error>
```

Acesse a aba Header da resposta, vá na propriedade **WWW-Authenticate** e procure por "Bearer realm=..." e guarde o código.

![realm](/sharepointConfig/imgs/Captura%20de%20tela%20de%202020-08-08%2018-10-37.png)

Pronto! Temos todos os dados necessário para autenticação.

## Configurando o ambiente no Insomnia

Agora vamos criar um ambiente no Insomnia e adicionar os nossos dados como variáveis:

```json
{
  "appReg_clientId": {{ clientID }},
  "appReg_clientSecret": {{ clientSecret }},
  "targetHost": "{{YourTenantName}}.sharepoint.com",
  "principal": "00000003-0000-0ff1-ce00-000000000000",
  "realm": {{ realmID }},
  "token": {{ geraremos a seguir... }}
}

```

> _Obs.: Não adicione o "https://" no **targetHost**, ele precisa estar parecido com isso: "delucagabriel.sharepoint.com"_

O código do Principal é referente ao Sharepoint, para obter mais informações, [vá neste artigo](https://blogs.msdn.microsoft.com/kaevans/2013/04/05/inside-sharepoint-2013-oauth-context-tokens/).

## Solicitando token

Depois que as variáveis ​​são configuradas, é hora de enviar uma solicitação POST para obter o token.

Crie uma nova solicitação no Insomnia, altere o tipo de solicitação para “POST”.

A URL será:
><https://accounts.accesscontrol.windows.net/{{realm}}/tokens/OAuth/2>

{{realm}} é uma variável de ambiente. Portanto, quando enviarmos a solicitação, {{realm}} será substituído pelo valor que especificamos anteriormente.

Clique na guia “Body”, escolha o formato "Form Url Encoded" e adicione os seguintes pares de chave-valor:

name | value
-----|------
grant_type | client_credentials
client_id | {{appReg_clientId}}@{{realm}}
client_secret | {{appReg_clientSecret}}
resource | {{principal}}/{{targetHost}}@{{realm}}

![Insomnia POST](/sharepointConfig/imgs/Captura%20de%20tela%20de%202020-08-08%2018-57-02.png)

![INSOMNIA](/sharepointConfig/imgs/Captura%20de%20tela%20de%202020-08-08%2019-00-27.png)

clique em "Send", copie o valor de "access_token", volte nas variáveis de ambiente do insomnia e coloque em token.
![POST](/sharepointConfig/imgs/Captura%20de%20tela%20de%202020-08-08%2019-00-48.png)

Pronto! Agora você pode efetuar suas chamadas para a API do Sharepoint.
