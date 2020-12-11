---
ms.openlocfilehash: 17c93f353c84ea2db28cd2e0203d30c5f320e36e
ms.sourcegitcommit: 141fe5c30dea84029ef61cf82558c35f2a744b65
ms.translationtype: MT
ms.contentlocale: es-ES
ms.lasthandoff: 12/11/2020
ms.locfileid: "49655236"
---
<!-- markdownlint-disable MD002 MD041 -->

<span data-ttu-id="f0baf-101">En este tutorial, creará una función de Azure sencilla que implementa las funciones desencadenadoras HTTP que llaman a Microsoft Graph.</span><span class="sxs-lookup"><span data-stu-id="f0baf-101">In this tutorial, you will create a simple Azure Function that implements HTTP trigger functions that call Microsoft Graph.</span></span> <span data-ttu-id="f0baf-102">Estas funciones tratarán los siguientes escenarios:</span><span class="sxs-lookup"><span data-stu-id="f0baf-102">These functions will cover the following scenarios:</span></span>

- <span data-ttu-id="f0baf-103">Implementa una API para obtener acceso a la bandeja de entrada de un usuario mediante la autenticación [en nombre de flujo](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-on-behalf-of-flow) .</span><span class="sxs-lookup"><span data-stu-id="f0baf-103">Implements an API to access a user's inbox using [on-behalf-of flow](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-on-behalf-of-flow) authentication.</span></span>
- <span data-ttu-id="f0baf-104">Implementa una API para suscribir y cancelar la suscripción a las notificaciones de la bandeja de entrada de un usuario, usando la autenticación del [flujo de concesión de credenciales de cliente](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) .</span><span class="sxs-lookup"><span data-stu-id="f0baf-104">Implements an API to subscribe and unsubscribe for notifications on a user's inbox, using using [client credentials grant flow](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) authentication.</span></span>
- <span data-ttu-id="f0baf-105">Implementa un webhook para recibir [notificaciones de cambios](https://docs.microsoft.com/graph/webhooks) de Microsoft Graph y tener acceso a los datos mediante el flujo de concesión de credenciales de cliente.</span><span class="sxs-lookup"><span data-stu-id="f0baf-105">Implements a webhook to receive [change notifications](https://docs.microsoft.com/graph/webhooks) from Microsoft Graph and access data using client credentials grant flow.</span></span>

<span data-ttu-id="f0baf-106">También se creará una aplicación de una sola página de JavaScript sencilla (SPA) para llamar a las API implementadas en la función de Azure.</span><span class="sxs-lookup"><span data-stu-id="f0baf-106">You will also create a simple JavaScript single-page application (SPA) to call the APIs implemented in the Azure Function.</span></span>

## <a name="create-azure-functions-project"></a><span data-ttu-id="f0baf-107">Crear proyecto de funciones de Azure</span><span class="sxs-lookup"><span data-stu-id="f0baf-107">Create Azure Functions project</span></span>

1. <span data-ttu-id="f0baf-108">Abra la interfaz de línea de comandos (CLI) en un directorio donde desee crear el proyecto.</span><span class="sxs-lookup"><span data-stu-id="f0baf-108">Open your command-line interface (CLI) in a directory where you want to create the project.</span></span> <span data-ttu-id="f0baf-109">Ejecute el comando siguiente.</span><span class="sxs-lookup"><span data-stu-id="f0baf-109">Run the following command.</span></span>

    ```Shell
    func init GraphTutorial --dotnet
    ```

1. <span data-ttu-id="f0baf-110">Cambie el directorio actual de la CLI al directorio **GraphTutorial** y ejecute los siguientes comandos para crear tres funciones en el proyecto.</span><span class="sxs-lookup"><span data-stu-id="f0baf-110">Change the current directory in your CLI to the **GraphTutorial** directory and run the following commands to create three functions in the project.</span></span>

    ```Shell
    func new --name GetMyNewestMessage --template "HTTP trigger" --language C#
    func new --name SetSubscription --template "HTTP trigger" --language C#
    func new --name Notify --template "HTTP trigger" --language C#
    ```

1. <span data-ttu-id="f0baf-111">Abra **local.settings.js** y agregue lo siguiente al archivo para permitir CORS en la `http://localhost:8080` dirección URL de la aplicación de prueba.</span><span class="sxs-lookup"><span data-stu-id="f0baf-111">Open **local.settings.json** and add the following to the file to allow CORS from `http://localhost:8080`, the URL for the test application.</span></span>

    ```json
    "Host": {
      "CORS": "http://localhost:8080"
    }
    ```

1. <span data-ttu-id="f0baf-112">Ejecute el siguiente comando para ejecutar el proyecto de forma local.</span><span class="sxs-lookup"><span data-stu-id="f0baf-112">Run the following command to run the project locally.</span></span>

    ```Shell
    func start
    ```

1. <span data-ttu-id="f0baf-113">Si todo funciona, verá el siguiente resultado:</span><span class="sxs-lookup"><span data-stu-id="f0baf-113">If everything is working, you will see the following output:</span></span>

    ```Shell
    Http Functions:

        GetMyNewestMessage: [GET,POST] http://localhost:7071/api/GetMyNewestMessage

        Notify: [GET,POST] http://localhost:7071/api/Notify

        SetSubscription: [GET,POST] http://localhost:7071/api/SetSubscription
    ```

1. <span data-ttu-id="f0baf-114">Para comprobar que las funciones funcionan correctamente, abra el explorador y busque las direcciones URL de la función que se muestran en el resultado.</span><span class="sxs-lookup"><span data-stu-id="f0baf-114">Verify that the functions are working correctly by opening your browser and browsing to the function URLs shown in the output.</span></span> <span data-ttu-id="f0baf-115">Debe ver el siguiente mensaje en el explorador: `This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.` .</span><span class="sxs-lookup"><span data-stu-id="f0baf-115">You should see the following message in your browser: `This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.`.</span></span>

## <a name="create-single-page-application"></a><span data-ttu-id="f0baf-116">Crear una aplicación de una sola página</span><span class="sxs-lookup"><span data-stu-id="f0baf-116">Create single-page application</span></span>

1. <span data-ttu-id="f0baf-117">Abra la CLI en un directorio en el que desee crear el proyecto.</span><span class="sxs-lookup"><span data-stu-id="f0baf-117">Open your CLI in a directory where you want to create the project.</span></span> <span data-ttu-id="f0baf-118">Cree un directorio denominado **TestClient** para contener los archivos HTML y JavaScript.</span><span class="sxs-lookup"><span data-stu-id="f0baf-118">Create a directory named **TestClient** to hold your HTML and JavaScript files.</span></span>

1. <span data-ttu-id="f0baf-119">Cree un nuevo archivo denominado **index.html** en el directorio **TestClient** y agregue el siguiente código.</span><span class="sxs-lookup"><span data-stu-id="f0baf-119">Create a new file named **index.html** in the **TestClient** directory and add the following code.</span></span>

    :::code language="html" source="../demo/TestClient/index.html" id="indexSnippet":::

    <span data-ttu-id="f0baf-120">Define el diseño básico de la aplicación, incluida una barra de navegación.</span><span class="sxs-lookup"><span data-stu-id="f0baf-120">This defines the basic layout of the app, including a navigation bar.</span></span> <span data-ttu-id="f0baf-121">También agrega lo siguiente:</span><span class="sxs-lookup"><span data-stu-id="f0baf-121">It also adds the following:</span></span>

    - <span data-ttu-id="f0baf-122">[Bootstrap](https://getbootstrap.com/) y JavaScript auxiliar</span><span class="sxs-lookup"><span data-stu-id="f0baf-122">[Bootstrap](https://getbootstrap.com/) and its supporting JavaScript</span></span>
    - [<span data-ttu-id="f0baf-123">FontAwesome</span><span class="sxs-lookup"><span data-stu-id="f0baf-123">FontAwesome</span></span>](https://fontawesome.com/)
    - [<span data-ttu-id="f0baf-124">Biblioteca de autenticación de Microsoft para JavaScript (MSAL.js) 2,0</span><span class="sxs-lookup"><span data-stu-id="f0baf-124">Microsoft Authentication Library for JavaScript (MSAL.js) 2.0</span></span>](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-browser)

    > [!TIP]
    > <span data-ttu-id="f0baf-125">La página incluye un favoritos, ( `<link rel="shortcut icon" href="g-raph.png">` ).</span><span class="sxs-lookup"><span data-stu-id="f0baf-125">The page includes a favicon, (`<link rel="shortcut icon" href="g-raph.png">`).</span></span> <span data-ttu-id="f0baf-126">Puede quitar esta línea o puede descargar el archivo de **g-raph.png** desde [GitHub](https://github.com/microsoftgraph/g-raph).</span><span class="sxs-lookup"><span data-stu-id="f0baf-126">You can remove this line, or you can download the **g-raph.png** file from [GitHub](https://github.com/microsoftgraph/g-raph).</span></span>

1. <span data-ttu-id="f0baf-127">Cree un nuevo archivo denominado **style. CSS** en el directorio **TestClient** y agregue el siguiente código.</span><span class="sxs-lookup"><span data-stu-id="f0baf-127">Create a new file named **style.css** in the **TestClient** directory and add the following code.</span></span>

    :::code language="css" source="../demo/TestClient/style.css":::

1. <span data-ttu-id="f0baf-128">Cree un nuevo archivo denominado **ui.js** en el directorio **TestClient** y agregue el siguiente código.</span><span class="sxs-lookup"><span data-stu-id="f0baf-128">Create a new file named **ui.js** in the **TestClient** directory and add the following code.</span></span>

    :::code language="javascript" source="../demo/TestClient/ui.js" id="uiJsSnippet":::

    <span data-ttu-id="f0baf-129">Este código usa JavaScript para representar la página actual en función de la vista seleccionada.</span><span class="sxs-lookup"><span data-stu-id="f0baf-129">This code uses JavaScript to render the current page based on the selected view.</span></span>

### <a name="test-the-single-page-application"></a><span data-ttu-id="f0baf-130">Probar la aplicación de una sola página</span><span class="sxs-lookup"><span data-stu-id="f0baf-130">Test the single-page application</span></span>

> [!NOTE]
> <span data-ttu-id="f0baf-131">En esta sección se incluyen instrucciones para usar [dotnet-servir](https://github.com/natemcmaster/dotnet-serve) para ejecutar un sencillo servidor HTTP de prueba en el equipo de desarrollo.</span><span class="sxs-lookup"><span data-stu-id="f0baf-131">This section includes instructions for using [dotnet-serve](https://github.com/natemcmaster/dotnet-serve) to run a simple testing HTTP server on your development machine.</span></span> <span data-ttu-id="f0baf-132">No es necesario usar esta herramienta específica.</span><span class="sxs-lookup"><span data-stu-id="f0baf-132">Using this specific tool is not required.</span></span> <span data-ttu-id="f0baf-133">Puede usar cualquier servidor de prueba que prefiera para servir el directorio **TestClient** .</span><span class="sxs-lookup"><span data-stu-id="f0baf-133">You can use any testing server you prefer to serve the **TestClient** directory.</span></span>

1. <span data-ttu-id="f0baf-134">Ejecute el siguiente comando en su CLI para instalar **dotnet-Serve**.</span><span class="sxs-lookup"><span data-stu-id="f0baf-134">Run the following command in your CLI to install **dotnet-serve**.</span></span>

    ```Shell
    dotnet tool install --global dotnet-serve
    ```

1. <span data-ttu-id="f0baf-135">Cambie el directorio actual de la CLI al directorio **TestClient** y ejecute el siguiente comando para iniciar un servidor http.</span><span class="sxs-lookup"><span data-stu-id="f0baf-135">Change the current directory in your CLI to the **TestClient** directory and run the following command to start an HTTP server.</span></span>

    ```Shell
    dotnet serve -h "Cache-Control: no-cache, no-store, must-revalidate" -p 8080
    ```

1. <span data-ttu-id="f0baf-136">Abra el explorador y vaya a `http://localhost:8080`.</span><span class="sxs-lookup"><span data-stu-id="f0baf-136">Open your browser and navigate to `http://localhost:8080`.</span></span> <span data-ttu-id="f0baf-137">La página debe representarse, pero ninguno de los botones funciona actualmente.</span><span class="sxs-lookup"><span data-stu-id="f0baf-137">The page should render, but none of the buttons currently work.</span></span>

## <a name="add-nuget-packages"></a><span data-ttu-id="f0baf-138">Agregar paquetes NuGet</span><span class="sxs-lookup"><span data-stu-id="f0baf-138">Add NuGet packages</span></span>

<span data-ttu-id="f0baf-139">Antes de continuar, instale algunos paquetes NuGet adicionales que usará más adelante.</span><span class="sxs-lookup"><span data-stu-id="f0baf-139">Before moving on, install some additional NuGet packages that you will use later.</span></span>

- <span data-ttu-id="f0baf-140">[Microsoft. Azure. functions. Extensions](https://www.nuget.org/packages/Microsoft.Azure.Functions.Extensions) para habilitar la inserción de dependencias en el proyecto de Azure functions.</span><span class="sxs-lookup"><span data-stu-id="f0baf-140">[Microsoft.Azure.Functions.Extensions](https://www.nuget.org/packages/Microsoft.Azure.Functions.Extensions) to enable dependency injection in the Azure Functions project.</span></span>
- <span data-ttu-id="f0baf-141">[Microsoft.Extensions.Configuration. UserSecrets](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.UserSecrets) para leer la configuración de la aplicación desde el [almacén secreto de desarrollo .net](https://docs.microsoft.com/aspnet/core/security/app-secrets).</span><span class="sxs-lookup"><span data-stu-id="f0baf-141">[Microsoft.Extensions.Configuration.UserSecrets](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.UserSecrets) to read application configuration from the [.NET development secret store](https://docs.microsoft.com/aspnet/core/security/app-secrets).</span></span>
- <span data-ttu-id="f0baf-142">[Microsoft. Graph](https://www.nuget.org/packages/Microsoft.Graph/) para realizar llamadas a Microsoft Graph.</span><span class="sxs-lookup"><span data-stu-id="f0baf-142">[Microsoft.Graph](https://www.nuget.org/packages/Microsoft.Graph/) for making calls to Microsoft Graph.</span></span>
- <span data-ttu-id="f0baf-143">[Microsoft. Identity. Client](https://www.nuget.org/packages/Microsoft.Identity.Client/) para autenticar y administrar tokens.</span><span class="sxs-lookup"><span data-stu-id="f0baf-143">[Microsoft.Identity.Client](https://www.nuget.org/packages/Microsoft.Identity.Client/) for authenticating and managing tokens.</span></span>
- <span data-ttu-id="f0baf-144">[Microsoft. IdentityModel. Protocols. OpenIdConnect](https://www.nuget.org/packages/Microsoft.IdentityModel.Protocols.OpenIdConnect) para recuperar la configuración de OpenID para la validación de tokens.</span><span class="sxs-lookup"><span data-stu-id="f0baf-144">[Microsoft.IdentityModel.Protocols.OpenIdConnect](https://www.nuget.org/packages/Microsoft.IdentityModel.Protocols.OpenIdConnect) for retrieving OpenID configuration for token validation.</span></span>
- <span data-ttu-id="f0baf-145">[System. IdentityModel. tokens. JWT](https://www.nuget.org/packages/System.IdentityModel.Tokens.Jwt) para validar tokens enviados a la API Web.</span><span class="sxs-lookup"><span data-stu-id="f0baf-145">[System.IdentityModel.Tokens.Jwt](https://www.nuget.org/packages/System.IdentityModel.Tokens.Jwt) for validating tokens sent to the web API.</span></span>

1. <span data-ttu-id="f0baf-146">Cambie el directorio actual de la CLI al directorio **GraphTutorial** y ejecute los siguientes comandos.</span><span class="sxs-lookup"><span data-stu-id="f0baf-146">Change the current directory in your CLI to the **GraphTutorial** directory and run the following commands.</span></span>

    ```Shell
    dotnet add package Microsoft.Azure.Functions.Extensions --version 1.0.0
    dotnet add package Microsoft.Extensions.Configuration.UserSecrets --version 3.1.5
    dotnet add package Microsoft.Graph --version 3.8.0
    dotnet add package Microsoft.Identity.Client --version 4.15.0
    dotnet add package Microsoft.IdentityModel.Protocols.OpenIdConnect --version 6.7.1
    dotnet add package System.IdentityModel.Tokens.Jwt --version 6.7.1
    ```
