<!-- default badges list -->
![](https://img.shields.io/endpoint?url=https://codecentral.devexpress.com/api/v1/VersionRange/626786332/23.2.3%2B)
[![](https://img.shields.io/badge/Open_in_DevExpress_Support_Center-FF7200?style=flat-square&logo=DevExpress&logoColor=white)](https://supportcenter.devexpress.com/ticket/details/T1159572)
[![](https://img.shields.io/badge/📖_How_to_use_DevExpress_Examples-e9f6fc?style=flat-square)](https://docs.devexpress.com/GeneralInformation/403183)
<!-- default badges end -->
# Role-Based Data Access with the DevExpress Web API Service

This example uses our free [.NET App Security & Web API Service](https://www.devexpress.com/products/net/application_framework/security-web-api-service.xml) to implement authentication and role-based data access. We ran a built-in wizard to generate a ready-to-use authentication service. This service uses [Entity Framework Core](https://docs.microsoft.com/en-us/ef/core/) to access a database. A .NET MAUI application sends requests to the Web API Service to obtain or modify data.

<img src="https://user-images.githubusercontent.com/12169834/228174231-0d1f2d88-8b2b-4db4-8969-fd9b2379d8ce.png" width="30%"/>

If you are new to the DevExpress .NET App Security & Web API Service, you may want to review the following resources:

[Create a Standalone Web API Application](https://docs.devexpress.com/eXpressAppFramework/403401/backend-web-api-service/create-new-application-with-web-api-service?p=net6)

[A 1-Click Solution for CRUD Web API with Role-based Access Control via EF Core & ASP.NET](https://www.youtube.com/watch?v=T7y4gwc1n4w&list=PL8h4jt35t1wiM1IOux04-8DiofuMEB33G)

## Prerequisites

[SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-downloads), if you run this solution on Windows.

## Run Projects

1. Run Visual Studio as Administrator and open the solution. Administrator privileges allow the IDE to create a database when you run the Web Service project. 

2. Select **WebApi** in the **debug** drop-down menu. This choice enables [Kestrel](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?view=aspnetcore-7.0) as the web server for debug runs.

    ![image](https://user-images.githubusercontent.com/12169834/231375597-9723a388-f8df-4e3b-ad42-135a150d2e78.png)

    If you prefer IIS Express to Kestrel, select **IIS Express** in the **debug** drop-down menu. Use an external text editor to add the following code to `.vs\MAUI_WebAPI\config\applicationhost.config`:

    ```xaml
    <sites>
        <site name="WebSite1" id="1" serverAutoStart="true">
        <!-* ... -->
            <bindings>
                <binding protocol="http" bindingInformation="*:65201:*" />
                <binding protocol="https" bindingInformation="*:44317:*" />
                <binding protocol="https" bindingInformation="*:44317:localhost" />
                <binding protocol="http" bindingInformation="*:65201:localhost" />
            </bindings>
        </site>
        <!-* ... -->
    </sites>
    ```

3. Right-click the `MAUI` project, choose `Set as Startup Project`, and select your emulator. Note that physical devices that are attached over USB cannot access your machine's localhost.
4. Right-click the `WebAPI` project and select `Debug > Start new instance`.
5. Right-click the `MAUI` project and select `Debug > Start new instance`.

## Implementation Details

### Service and Communication

* DevExpress Web API Service uses JSON Web Tokens (JWT) to authorize users. To obtain a token, pass username and password to the **Authenticate** endpoint. In this example, token generation logic is implemented in the `WebAPIService.RequestTokenAsync` method:

    ```csharp
      private async Task<HttpResponseMessage> RequestTokenAsync(string userName, string password) {
            return await HttpClient.PostAsync($"{ApiUrl}Authentication/Authenticate",
                                                new StringContent(JsonSerializer.Serialize(new { userName, password = $"{password}" }), Encoding.UTF8,
                                                ApplicationJson));
      }
    ```

    Include the token in [HttpClient.DefaultRequestHeaders.Authorization](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.headers.httprequestheaders.authorization?view=net-7.0). All subsequent requests can then access private endpoints and data: 

    ```csharp
    HttpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", await tokenResponse.Content.ReadAsStringAsync());
    ```

  File to Look At: [WebAPIService.cs](CS/MAUI/Services/WebAPIService.cs)

* We implemented the following custom endpoints in the `WebApi` service:

    * The **CanDeletePost** endpoint allows you to send a request from a mobile device to the service and check whether the current user can delete posts. This allows you to show/hide the delete button in the UI.

        File to Look At: [Updater.cs](CS/WebApi/Controllers/CustomEndpointController.cs)

    * The **CurrentUser** endpoint returns information about the authenticated user.

        File to Look At: [Updater.cs](CS/WebApi/Controllers/CustomEndpointController.cs)

    * The **GetAuthorImage** endpoint retrieves an author image by user ID. 

        File to Look At: [Updater.cs](CS/WebApi/Controllers/PublicEndpointController.cs)

    * The **GetPostImage** endpoint retrieves an image by post ID. 

        File to Look At: [Updater.cs](CS/WebApi/Controllers/PublicEndpointController.cs)


* The `Updater.UpdateDatabaseAfterUpdateSchema` method generates users and specifies their login credentials. You can modify a user's password directly in the database. Note: Our cross-platform .NET Application Framework ([XAF UI](https://docs.devexpress.com/eXpressAppFramework/112649/data-security-and-safety/security-system/authentication/passwords-in-the-security-system)) allows you to quickly build a desktop or web UI that accesses the same database.

    File to Look At: [Updater.cs](CS/WebApi/DatabaseUpdate/Updater.cs)

* `PermissionPolicyRole` objects in the `Updater` class add user permissions. The following code snippet calls the `AddObjectPermissionFromLambda` method to configure the "Viewer" role (allow the user to read published posts):

    ```csharp
    role.AddObjectPermissionFromLambda(SecurityOperations.Read, p => p.IsPublished, SecurityPermissionState.Allow);
    ```

    File to Look At: [Updater.cs](CS/WebApi/DatabaseUpdate/Updater.cs)

* The `AddTypePermissionsRecursively` method modifies privileges for the "Editor" role (Alex, Antony, and Dennis). The method adds CRUD permissions (create, read, update, delete) for the `Post` type:
    
    ```csharp
    role.AddTypePermissionsRecursively<Post>(SecurityOperations.Read | SecurityOperations.Write | SecurityOperations.Create | SecurityOperations.DeleteObject, SecurityPermissionState.Allow);
    ```

    File to Look At: [Updater.cs](CS/WebApi/DatabaseUpdate/Updater.cs)

### Login UI and View Model

* Use [TextEdit.StartIcon](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.EditBase.StartIcon) and [PasswordEdit.StartIcon](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.EditBase.StartIcon) properties to display icons in [TextEdit](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.TextEdit) and [PasswordEdit](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.PasswordEdit) controls.

    ```xaml
    <dxe:TextEdit LabelText="Login" StartIcon="editorsname" .../>
    <dxe:PasswordEdit LabelText="Password" StartIcon="editorspassword" .../>
    ```

    File to Look At: [LoginPage.xaml](CS/MAUI/Views/LoginPage.xaml)
    
* To validate user input in the [PasswordEdit](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.PasswordEdit) control, use [EditBase.HasError](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.EditBase.HasError) and [EditBase.ErrorText](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.EditBase.ErrorText) inherited properties.

    ```xaml
    <dxe:PasswordEdit ... HasError="{Binding HasError}" ErrorText="{Binding ErrorText}"/>
    ```

    File to Look At: [LoginPage.xaml](CS/MAUI/Views/LoginPage.xaml)

    ```csharp
    public class LoginViewModel : BaseViewModel {
        // ...
        string errorText;
        bool hasError;
        // ...

        public string ErrorText {
            get => errorText;
            set => SetProperty(ref errorText, value);
        }

        public bool HasError {
            get => hasError;
            set => SetProperty(ref hasError, value);
        }
        async void OnLoginClicked() {
            /// ...
            string response = await DataStore.Authenticate(userName, password);
            if (!string.IsNullOrEmpty(response)) {
                ErrorText = response;
                HasError = true;
                return;
            }
            HasError = false;
            await Navigation.NavigateToAsync<SuccessViewModel>();
        }
    }
    ```

    File to Look At: [LoginViewModel.cs](CS/MAUI/ViewModels/LoginViewModel.cs)

* Specify the [TextEdit.ReturnType](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.EditBase.ReturnType) inherited property to focus the [PasswordEdit](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.PasswordEdit) control after the [TextEdit](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.TextEdit) control's value is edited.
* Use the [PasswordEdit.ReturnCommand](https://docs.devexpress.com/MAUI/DevExpress.Maui.Editors.EditBase.ReturnCommand) property to specify a command (**Login**) that runs when a user enters the password:

    ```xaml
    <dxe:PasswordEdit ReturnCommand="{Binding LoginCommand}"/>
    ```

    File to Look At: [LoginPage.xaml](CS/MAUI/Views/LoginPage.xaml)
    
    ```csharp
    public class LoginViewModel : BaseViewModel {
        // ...
        public LoginViewModel() {
            LoginCommand = new Command(OnLoginClicked);
            SignUpCommand = new Command(OnSignUpClicked);
            PropertyChanged +=
                (_, __) => LoginCommand.ChangeCanExecute();

        }
        // ...
        public Command LoginCommand { get; }
        public Command SignUpCommand { get; }
        // ...
    }
    ```

    File to Look At: [LoginViewModel.cs](CS/MAUI/ViewModels/LoginViewModel.cs)

* We enabled image caching in this project. To achieve that, we needed to identify images by their Uri. To create a Uri, we use a [MultiBinding](https://learn.microsoft.com/en-us/dotnet/maui/fundamentals/data-binding/multibinding?view=net-maui-7.0) that obtains the host name and the author/post ID. For additional information on image caching, refer to [MAUI documentation](https://learn.microsoft.com/en-us/dotnet/maui/user-interface/controls/image?view=net-maui-7.0#load-a-remote-image). 

    ```xaml
    <Image>
        <Image.Source>
            <MultiBinding StringFormat="{}{0}PublicEndpoint/PostImage/{1}">
                <Binding Source="{x:Static webService:WebAPIService.ApiUrl}"/>
                <Binding Path="PostId"/>
            </MultiBinding>
        </Image.Source>
    </Image>
    ```

    File to Look At: [ItemsPage.xaml](CS/MAUI/Views/ItemsPage.xaml)


## Debug Specifics

Android emulator and iOS simulator request a certificate to access a service over HTTPS. In this example, we switch to HTTP in debug mode:

```csharp
#if !DEBUG
    app.UseHttpsRedirection();
#endif
```

#### MAUI - Android

```xml
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">10.0.2.2</domain>
    </domain-config>
</network-security-config>
```

#### MAUI - iOS

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsLocalNetworking</key>
    <true/>
</dict>
```

This allows you to bypass the certificate check without the need to create a development certificate or implement HttpClient handlers.

For more information, please refer to [Connect to local web services from Android emulators and iOS simulators](https://learn.microsoft.com/en-us/dotnet/maui/data-cloud/local-web-services?view=net-maui-7.0#android-network-security-configuration).

**We recommend that you use HTTP only when you develop/debug your application. In production, use HTTPS for security reasons.**

## Files to Look At

* [WebAPIService.cs](CS/MAUI/Services/WebAPIService.cs)
* [Updater.cs](CS/WebApi/DatabaseUpdate/Updater.cs)
* [LoginPage.xaml](CS/MAUI/Views/LoginPage.xaml)
* [ItemsPage.xaml](CS/MAUI/Views/ItemsPage.xaml)
* [LoginViewModel.cs](CS/MAUI/ViewModels/LoginViewModel.cs)

## Documentation

* [Featured Scenario: Role-Based Data Access](https://docs.devexpress.com/MAUI/404316)
* [Featured Scenarios](https://docs.devexpress.com/MAUI/404291)
* [Create a Standalone Web API Application](https://docs.devexpress.com/eXpressAppFramework/403401/backend-web-api-service/create-new-application-with-web-api-service?p=net6)

## More Examples

* [How to Create a Web API Service Backend for a .NET MAUI Application](https://www.devexpress.com/go/XAF_Security_NonXAF_MAUI.aspx)
* [Authenticate Users with the Web API Service](https://github.com/DevExpress-Examples/maui-authenticate/)
* [DevExpress Mobile UI for .NET MAUI](https://github.com/DevExpress-Examples/maui-demo-app/)
