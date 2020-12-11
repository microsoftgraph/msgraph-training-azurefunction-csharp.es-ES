---
ms.openlocfilehash: eb227079656e2a57550511c3abfacb49935fe46a
ms.sourcegitcommit: 141fe5c30dea84029ef61cf82558c35f2a744b65
ms.translationtype: MT
ms.contentlocale: es-ES
ms.lasthandoff: 12/11/2020
ms.locfileid: "49655243"
---
<!-- markdownlint-disable MD002 MD041 -->

En este ejercicio finalizará la implementación de las funciones de Azure `SetSubscription` y se `Notify` actualizará la aplicación de prueba para suscribirse y cancelar la suscripción a los cambios en la bandeja de entrada de un usuario.

- La `SetSubscription` función actuará como una API, lo que permite que la aplicación de prueba cree o elimine una [suscripción](https://docs.microsoft.com/graph/webhooks) a los cambios en la bandeja de entrada de un usuario.
- La `Notify` función actuará como el webhook que recibe las notificaciones de cambios generadas por la suscripción.

Ambas funciones usarán el [flujo de concesión de credenciales de cliente](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) para obtener un token de solo aplicación para llamar a Microsoft Graph. Debido a que un administrador concedió el consentimiento del administrador a los ámbitos de permiso necesarios, no se requiere ninguna interacción del usuario para obtener el token.

## <a name="add-client-credentials-authentication-to-the-azure-functions-project"></a>Agregar autenticación de credenciales de cliente al proyecto de funciones de Azure

En esta sección, implementará el flujo de credenciales de cliente en el proyecto de funciones de Azure para obtener un token de acceso compatible con Microsoft Graph.

1. Abra su CLI en el directorio que contiene **GraphTutorial. csproj**.

1. Agregue el identificador y el secreto de la aplicación de webhook al almacén secreto mediante los comandos siguientes. Reemplace `YOUR_WEBHOOK_APP_ID_HERE` por el identificador de la aplicación para el **webhook** de la función de Microsoft de Graph. Reemplace `YOUR_WEBHOOK_APP_SECRET_HERE` por el secreto de aplicación que ha creado en Azure portal para el **webhook de la función Graph de Azure**.

    ```Shell
    dotnet user-secrets set webHookId "YOUR_WEBHOOK_APP_ID_HERE"
    dotnet user-secrets set webHookSecret "YOUR_WEBHOOK_APP_SECRET_HERE"
    ```

### <a name="create-a-client-credentials-authentication-provider"></a>Crear un proveedor de autenticación de credenciales de cliente

1. Cree un archivo nuevo en el directorio **./GraphTutorial/Authentication** denominado **ClientCredentialsAuthProvider.CS** y agregue el siguiente código.

    :::code language="csharp" source="../demo/GraphTutorial/Authentication/ClientCredentialsAuthProvider.cs" id="AuthProviderSnippet":::

Tómese un momento para considerar lo que hace el código en **ClientCredentialsAuthProvider.CS** .

- En el constructor, Inicializa un **ConfidentialClientApplication** desde el `Microsoft.Identity.Client` paquete. Usa las `WithAuthority(AadAuthorityAudience.AzureAdMyOrg, true)` funciones y `.WithTenantId(tenantId)` para restringir la audiencia de inicio de sesión a solo la organización de Microsoft 365 especificada.
- En la `GetAccessToken` función, llama `AcquireTokenForClient` a para obtener un token para la aplicación. El flujo de tokens de credenciales de cliente es siempre no interactivo.
- Implementa la `Microsoft.Graph.IAuthenticationProvider` interfaz, lo que permite pasar esta clase en el constructor de `GraphServiceClient` para autenticar las solicitudes salientes.

## <a name="update-graphclientservice"></a>Actualizar GraphClientService

1. Abra **GraphClientService.CS** y agregue la siguiente propiedad a la clase.

    :::code language="csharp" source="../demo/GraphTutorial/Services/GraphClientService.cs" id="AppGraphClientMembers":::

1. Reemplace la función `GetAppGraphClient` existente por lo siguiente.

    :::code language="csharp" source="../demo/GraphTutorial/Services/GraphClientService.cs" id="AppGraphClientFunctions":::

## <a name="implement-notify-function"></a>Función de implementación de notificación

En esta sección, implementará la `Notify` función, que se utilizará como dirección URL de notificación para las notificaciones de cambios.

1. Cree un nuevo directorio en el directorio **GraphTutorials** denominado **Models**.

1. Cree un nuevo archivo en el directorio **modelos** denominado **ResourceData.CS** y agregue el siguiente código.

    :::code language="csharp" source="../demo/GraphTutorial/Models/ResourceData.cs" id="ResourceDataSnippet":::

1. Cree un nuevo archivo en el directorio **modelos** denominado **ChangeNotification.CS** y agregue el siguiente código.

    :::code language="csharp" source="../demo/GraphTutorial/Models/ChangeNotification.cs" id="ChangeNotificationSnippet":::

1. Cree un nuevo archivo en el directorio **modelos** denominado **NotificationList.CS** y agregue el siguiente código.

    :::code language="csharp" source="../demo/GraphTutorial/Models/NotificationList.cs" id="NotificationListSnippet":::

1. Abra **./GraphTutorial/Notify.CS** y reemplace todo el contenido por lo siguiente.

    :::code language="csharp" source="../demo/GraphTutorial/Notify.cs" id="NotifySnippet":::

Tómese un momento para considerar lo que hace el código en **Notify.CS** .

- La `Run` función comprueba la presencia de un `validationToken` parámetro de consulta. Si ese parámetro está presente, procesa la solicitud como una [solicitud de validación](https://docs.microsoft.com/graph/webhooks#notification-endpoint-validation)y responde en consecuencia.
- Si la solicitud no es una solicitud de validación, la carga JSON se deserializa en un `NotificationList` .
- Cada notificación de la lista se comprueba para el valor de estado de cliente esperado y se procesa.
- El mensaje que desencadenó la notificación se recupera con Microsoft Graph.

## <a name="implement-setsubscription-function"></a>Implementar la función SetSubscription

En esta sección, implementará la función SetSubscription. Esta función actuará como una API a la que llama la aplicación de prueba para crear o eliminar una suscripción en la bandeja de entrada de un usuario.

1. Cree un nuevo archivo en el directorio **modelos** denominado **SetSubscriptionPayload.CS** y agregue el siguiente código.

    :::code language="csharp" source="../demo/GraphTutorial/Models/SetSubscriptionPayload.cs" id="SetSubscriptionPayloadSnippet":::

1. Abra **./GraphTutorial/SetSubscription.CS** y reemplace todo el contenido por lo siguiente.

    :::code language="csharp" source="../demo/GraphTutorial/SetSubscription.cs" id="SetSubscriptionSnippet":::

Tómese un momento para considerar lo que hace el código en **SetSubscription.CS** .

- La `Run` función lee la carga útil JSON enviada en la solicitud post para determinar el tipo de solicitud (subscribe o unsubscribe), el identificador de usuario para suscribirse y el identificador de suscripción para cancelar la suscripción.
- Si la solicitud es una solicitud subscribe, usa el SDK de Microsoft Graph para crear una nueva suscripción en la bandeja de entrada del usuario especificado. La suscripción recibirá una notificación cuando se creen o actualicen los mensajes. La nueva suscripción se devuelve en la carga JSON de la respuesta.
- Si la solicitud es una solicitud de cancelación de suscripción, usa el SDK de Microsoft Graph para eliminar la suscripción especificada.

## <a name="call-setsubscription-from-the-test-app"></a>Llamar a SetSubscription desde la aplicación de prueba

En esta sección, implementará funciones para crear y eliminar suscripciones en la aplicación de prueba.

1. Abra el **azurefunctions.js./TestClient/** y agregue la siguiente función.

    :::code language="javascript" source="../demo/TestClient/azurefunctions.js" id="createSubscriptionSnippet":::

    Este código llama a la `SetSubscription` función Azure para suscribirse y agrega la nueva suscripción a la matriz de suscripciones de la sesión.

1. Agregue la siguiente función a **azurefunctions.js**.

    :::code language="javascript" source="../demo/TestClient/azurefunctions.js" id="deleteSubscriptionSnippet":::

    Este código llama `SetSubscription` a la función de Azure para anular y quitar la suscripción de la matriz de suscripciones de la sesión.

1. Si no se está ejecutando ngrok, ejecute ngrok ( `ngrok http 7071` ) y copie la dirección URL de reenvío https.

1. Para agregar la dirección URL ngrok al almacén de secretos de usuario, ejecute el siguiente comando.

    ```Shell
    dotnet user-secrets set ngrokUrl "YOUR_NGROK_URL_HERE"
    ```

    > [!IMPORTANT]
    > Si reinicia ngrok, tendrá que repetir este comando para actualizar la dirección URL de ngrok.

1. Cambie el directorio actual de la CLI al directorio **./GraphTutorial** y ejecute el siguiente comando para iniciar la función de Azure de forma local.

    ```Shell
    func start
    ```

1. Actualice el SPA y seleccione el elemento **suscripciones** NAV. Escriba un identificador de usuario para un usuario de la organización de Microsoft 365 que tenga un buzón de correo de Exchange Online. Puede ser el del usuario `id` (de Microsoft Graph) o el del usuario `userPrincipalName` . Haga clic en **suscribirse**.

1. La página se actualizará y mostrará la nueva suscripción en la tabla.

1. Enviar un correo electrónico al usuario. Después de un breve período de tiempo, `Notify` se debe llamar a la función. Puede comprobarlo en la interfaz Web de ngrok ( `http://localhost:4040` ) o en el resultado de la depuración del proyecto de la función de Azure.

    ```Shell
    ...
    [7/8/2020 7:33:57 PM] The following message was created:
    [7/8/2020 7:33:57 PM] Subject: Hi Megan!, ID: AAMkAGUyN2I4N2RlLTEzMTAtNDBmYy1hODdlLTY2NTQwODE2MGEwZgBGAAAAAAA2J9QH-DvMRK3pBt_8rA6nBwCuPIFjbMEkToHcVnQirM5qAAAAAAEMAACuPIFjbMEkToHcVnQirM5qAACHmpAsAAA=
    [7/8/2020 7:33:57 PM] Executed 'Notify' (Succeeded, Id=9c40af0b-e082-4418-aa3a-aee624f30e7a)
    ...
    ```

1. En la aplicación de prueba, haga clic en **eliminar** en la fila de la tabla de la suscripción. La página se actualiza y la suscripción ya no está en la tabla.
