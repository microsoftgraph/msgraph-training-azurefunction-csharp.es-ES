---
ms.openlocfilehash: 17c93f353c84ea2db28cd2e0203d30c5f320e36e
ms.sourcegitcommit: 141fe5c30dea84029ef61cf82558c35f2a744b65
ms.translationtype: MT
ms.contentlocale: es-ES
ms.lasthandoff: 12/11/2020
ms.locfileid: "49655236"
---
<!-- markdownlint-disable MD002 MD041 -->

En este tutorial, creará una función de Azure sencilla que implementa las funciones desencadenadoras HTTP que llaman a Microsoft Graph. Estas funciones tratarán los siguientes escenarios:

- Implementa una API para obtener acceso a la bandeja de entrada de un usuario mediante la autenticación [en nombre de flujo](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-on-behalf-of-flow) .
- Implementa una API para suscribir y cancelar la suscripción a las notificaciones de la bandeja de entrada de un usuario, usando la autenticación del [flujo de concesión de credenciales de cliente](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) .
- Implementa un webhook para recibir [notificaciones de cambios](https://docs.microsoft.com/graph/webhooks) de Microsoft Graph y tener acceso a los datos mediante el flujo de concesión de credenciales de cliente.

También se creará una aplicación de una sola página de JavaScript sencilla (SPA) para llamar a las API implementadas en la función de Azure.

## <a name="create-azure-functions-project"></a>Crear proyecto de funciones de Azure

1. Abra la interfaz de línea de comandos (CLI) en un directorio donde desee crear el proyecto. Ejecute el comando siguiente.

    ```Shell
    func init GraphTutorial --dotnet
    ```

1. Cambie el directorio actual de la CLI al directorio **GraphTutorial** y ejecute los siguientes comandos para crear tres funciones en el proyecto.

    ```Shell
    func new --name GetMyNewestMessage --template "HTTP trigger" --language C#
    func new --name SetSubscription --template "HTTP trigger" --language C#
    func new --name Notify --template "HTTP trigger" --language C#
    ```

1. Abra **local.settings.js** y agregue lo siguiente al archivo para permitir CORS en la `http://localhost:8080` dirección URL de la aplicación de prueba.

    ```json
    "Host": {
      "CORS": "http://localhost:8080"
    }
    ```

1. Ejecute el siguiente comando para ejecutar el proyecto de forma local.

    ```Shell
    func start
    ```

1. Si todo funciona, verá el siguiente resultado:

    ```Shell
    Http Functions:

        GetMyNewestMessage: [GET,POST] http://localhost:7071/api/GetMyNewestMessage

        Notify: [GET,POST] http://localhost:7071/api/Notify

        SetSubscription: [GET,POST] http://localhost:7071/api/SetSubscription
    ```

1. Para comprobar que las funciones funcionan correctamente, abra el explorador y busque las direcciones URL de la función que se muestran en el resultado. Debe ver el siguiente mensaje en el explorador: `This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.` .

## <a name="create-single-page-application"></a>Crear una aplicación de una sola página

1. Abra la CLI en un directorio en el que desee crear el proyecto. Cree un directorio denominado **TestClient** para contener los archivos HTML y JavaScript.

1. Cree un nuevo archivo denominado **index.html** en el directorio **TestClient** y agregue el siguiente código.

    :::code language="html" source="../demo/TestClient/index.html" id="indexSnippet":::

    Define el diseño básico de la aplicación, incluida una barra de navegación. También agrega lo siguiente:

    - [Bootstrap](https://getbootstrap.com/) y JavaScript auxiliar
    - [FontAwesome](https://fontawesome.com/)
    - [Biblioteca de autenticación de Microsoft para JavaScript (MSAL.js) 2,0](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-browser)

    > [!TIP]
    > La página incluye un favoritos, ( `<link rel="shortcut icon" href="g-raph.png">` ). Puede quitar esta línea o puede descargar el archivo de **g-raph.png** desde [GitHub](https://github.com/microsoftgraph/g-raph).

1. Cree un nuevo archivo denominado **style. CSS** en el directorio **TestClient** y agregue el siguiente código.

    :::code language="css" source="../demo/TestClient/style.css":::

1. Cree un nuevo archivo denominado **ui.js** en el directorio **TestClient** y agregue el siguiente código.

    :::code language="javascript" source="../demo/TestClient/ui.js" id="uiJsSnippet":::

    Este código usa JavaScript para representar la página actual en función de la vista seleccionada.

### <a name="test-the-single-page-application"></a>Probar la aplicación de una sola página

> [!NOTE]
> En esta sección se incluyen instrucciones para usar [dotnet-servir](https://github.com/natemcmaster/dotnet-serve) para ejecutar un sencillo servidor HTTP de prueba en el equipo de desarrollo. No es necesario usar esta herramienta específica. Puede usar cualquier servidor de prueba que prefiera para servir el directorio **TestClient** .

1. Ejecute el siguiente comando en su CLI para instalar **dotnet-Serve**.

    ```Shell
    dotnet tool install --global dotnet-serve
    ```

1. Cambie el directorio actual de la CLI al directorio **TestClient** y ejecute el siguiente comando para iniciar un servidor http.

    ```Shell
    dotnet serve -h "Cache-Control: no-cache, no-store, must-revalidate" -p 8080
    ```

1. Abra el explorador y vaya a `http://localhost:8080`. La página debe representarse, pero ninguno de los botones funciona actualmente.

## <a name="add-nuget-packages"></a>Agregar paquetes NuGet

Antes de continuar, instale algunos paquetes NuGet adicionales que usará más adelante.

- [Microsoft. Azure. functions. Extensions](https://www.nuget.org/packages/Microsoft.Azure.Functions.Extensions) para habilitar la inserción de dependencias en el proyecto de Azure functions.
- [Microsoft.Extensions.Configuration. UserSecrets](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.UserSecrets) para leer la configuración de la aplicación desde el [almacén secreto de desarrollo .net](https://docs.microsoft.com/aspnet/core/security/app-secrets).
- [Microsoft. Graph](https://www.nuget.org/packages/Microsoft.Graph/) para realizar llamadas a Microsoft Graph.
- [Microsoft. Identity. Client](https://www.nuget.org/packages/Microsoft.Identity.Client/) para autenticar y administrar tokens.
- [Microsoft. IdentityModel. Protocols. OpenIdConnect](https://www.nuget.org/packages/Microsoft.IdentityModel.Protocols.OpenIdConnect) para recuperar la configuración de OpenID para la validación de tokens.
- [System. IdentityModel. tokens. JWT](https://www.nuget.org/packages/System.IdentityModel.Tokens.Jwt) para validar tokens enviados a la API Web.

1. Cambie el directorio actual de la CLI al directorio **GraphTutorial** y ejecute los siguientes comandos.

    ```Shell
    dotnet add package Microsoft.Azure.Functions.Extensions --version 1.0.0
    dotnet add package Microsoft.Extensions.Configuration.UserSecrets --version 3.1.5
    dotnet add package Microsoft.Graph --version 3.8.0
    dotnet add package Microsoft.Identity.Client --version 4.15.0
    dotnet add package Microsoft.IdentityModel.Protocols.OpenIdConnect --version 6.7.1
    dotnet add package System.IdentityModel.Tokens.Jwt --version 6.7.1
    ```
