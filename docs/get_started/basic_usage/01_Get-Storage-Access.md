
# Get Storage Access

## Introduction

The preparation for creating an account in Aruna is minimal. It is only required that you are already registered in a supported AAI. Currently that is: 

* **GWDG SSO Service**, i.e. DFN AAI, Life Science Login (ELIXIR AAI) or GfBio Accounts
* **Direct LifeScience RI Login**


These are also the current options that are offered if you want to register or login via the [Aruna demo website](https://dev.aruna-engine.org/){:target="_blank"}. If you want to register via the Aruna API instead, you just have to put the OIDC token you received from one of the previously mentioned services into the [user registration request header](#user-registration) for authorization.

From here on you have two possibilities to authenticate/authorize all of your actions inside Aruna system:

1. You can use your OIDC token which consumes the granted permissions you have on resources.
2. You create API tokens which can either 
    * Consume the permissions you have on resources
    * Have scoped permissions which are directly associated with the token


## Client creation

If you plan to send your requests via gRPC the first step is to establish a connection to the gRPC gateway of the server you want to send the requests to.

The presence of a client connection to the specific resource service is required for all further requests in this tutorial if the requests are send via gRPC.

!!! Info "Development instance endpoint URLs"

    The development instance can be used for testing purposes. It can be accessed via the following URLs after successful registration.

    **Please remember that the development instance in no way guarantees data consistency and availability!**

    * JSON-over-HTTP: `https://api.dev.aruna-engine.org` or localized e.g. `https://api.gi.dev.aruna-engine.org`
    * gRPC Clients: `https://grpc.dev.aruna-engine.org` or localized e.g. `https://grpc.gi.dev.aruna-engine.org`
    * DataProxies: e.g. `https://proxy.gi.dev.aruna-engine.org`

    **It is also emphasized that Aruna is a data orchestration engine that orchestrates data and metadata on behalf of multiple storage instances.<br/>
    While we provide some physical storage for our partners, not all storage instances are operated by us.**

!!! Danger

    These are minimal reproducible examples only for demonstration purposes which should not be used _'as-is'_ in a production environment!

=== ":simple-rust: Rust"

    To use the Rust API library you have to set it as dependency `aruna-rust-api = "<Aruna-Rust-API-Version>"` in the `cargo.toml` of your project.

    You can find the latest version of the Aruna Rust API package on [crates.io](https://crates.io/crates/aruna-rust-api){:target="_blank"}.

    !!! note
    
        This example additionally implements an interceptor which adds the authorization token automatically to each request send by the service clients.

        It is a little more complicated and extra work up front but offers the advantage of having to worry less about the correct API token request metadata later.

        **All Rust examples in this documentation assume that the client services have been initialized with an interceptor.**

    ```rust linenums="1"
    use aruna_rust_api::api::storage::services::v2::{
        authorization_service_client::AuthorizationServiceClient,
        collection_service_client::CollectionServiceClient,
        dataset_service_client::DatasetServiceClient,
        object_service_client::ObjectServiceClient,
        project_service_client::ProjectServiceClient,
        storage_status_service_client::StorageStatusServiceClient,
        user_service_client::UserServiceClient,
    }
    use std::sync::Arc;
    use tonic::codegen::InterceptedService;
    use tonic::metadata::{AsciiMetadataKey, AsciiMetadataValue};
    use tonic::transport::{Channel, ClientTlsConfig};
    
    // Create a client interceptor which always adds the specified api token to the request header
    #[derive(Clone)]
    pub struct ClientInterceptor {
        api_token: String,
    }
    // Implement a request interceptor which always adds 
    //  the authorization header with a specific API token to all requests
    impl tonic::service::Interceptor for ClientInterceptor {
        fn call(&mut self, request: tonic::Request<()>) -> Result<tonic::Request<()>, tonic::Status> {
            let mut mut_req: tonic::Request<()> = request;
            let metadata = mut_req.metadata_mut();
            metadata.append(
                AsciiMetadataKey::from_bytes("Authorization".as_bytes()).unwrap(),
                AsciiMetadataValue::try_from(format!("Bearer {}", self.api_token.as_str())).unwrap(),
            );
    
            return Ok(mut_req);
        }
    }
    
    fn main() {
        // Create connection to the Aruna instance via gRPC
        let api_token   = "MySecretArunaApiToken".to_string();
        let tls_config  = ClientTlsConfig::new();
        let endpoint    = Channel::from_shared("https://<URL-To-Aruna-Instance-gRPC-endpoint>").unwrap().tls_config(tls_config).unwrap();
        let channel     = endpoint.connect().await.unwrap();
        let interceptor = ClientInterceptor { api_token: api_token.clone() };
    
        // Create the individual client services
        let mut info_client       = StorageStatusServiceClient::new(channel.clone());
        let mut auth_client       = AuthorizationServiceClient::with_interceptor(channel.clone(), interceptor.clone());
        let mut user_client       = UserServiceClient::with_interceptor(channel.clone(), interceptor.clone());
        let mut project_client    = ProjectServiceClient::with_interceptor(channel.clone(), interceptor.clone());
        let mut collection_client = CollectionServiceClient::with_interceptor(channel.clone(), interceptor.clone());
        let mut dataset_client    = DatasetServiceClient::with_interceptor(channel.clone(), interceptor.clone());
        let mut object_client     = ObjectServiceClient::with_interceptor(channel.clone(), interceptor.clone());
        // let mut other_client = ...
        
        // Do something with the client services ...
    }
    ```

=== ":simple-python: Python"

    To use the Python API library in your Python project you have to install the [PyPI package](https://pypi.org/project/Aruna-Python-API/){:target="_blank"}: `pip install Aruna-Python-API`.

    !!! note
    
        This example additionally implements an interceptor which adds the authorization token automatically to each request send by the service clients.

        It is a little more complicated and extra work up front but offers the advantage of having to worry less about the correct API token request metadata later.

        **All Python examples in this documentation assume that the client services have been initialized with an interceptor.**

    ```python linenums="1"
    import collections
    import grpc

    from aruna.api.storage.services.v2.info_service_pb2_grpc import StorageStatusServiceStub
    from aruna.api.storage.services.v2.authorization_service_pb2_grpc import AuthorizationServiceStub
    from aruna.api.storage.services.v2.user_service_pb2_grpc import UserServiceStub
    from aruna.api.storage.services.v2.project_service_pb2_grpc import ProjectServiceStub
    from aruna.api.storage.services.v2.collection_service_pb2_grpc import CollectionServiceStub
    from aruna.api.storage.services.v2.dataset_service_pb2_grpc import DatasetServiceStub
    from aruna.api.storage.services.v2.object_service_pb2_grpc import ObjectServiceStub
    
    # Valid Aruna API token
    #   In a production environment this should be stored in a more secure location ...
    API_TOKEN = 'MySecretArunaApiToken'
    
    # Aruna instance gRPC endpoint
    ARUNA_HOST = '<URL-To-Aruna-Instance-gRPC-endpoint>' # Protocol (e.g. https://) has to be omitted
    ARUNA_PORT = '443'


    class _MyAuthInterceptor(grpc.UnaryUnaryClientInterceptor):
        """
        Implement abstract class grpc.UnaryUnaryClientInterceptor to extend request metadata.
        """
        def intercept_unary_unary(self, continuation, client_call_details, request):
            # Append authorization token to request metadata
            metadata = []
            if client_call_details.metadata is not None:
               metadata = list(client_call_details.metadata)
            metadata.append(('authorization', f'Bearer {API_TOKEN}'))
    
            # Continue with new client call details
            request_iterator = iter((request,))
            updated_details = _ClientCallDetails(
                client_call_details.method, client_call_details.timeout, 
                metadata, client_call_details.credentials
            )
    
            return continuation(updated_details, next(request_iterator))
    
    
    class _ClientCallDetails(
            collections.namedtuple(
                '_ClientCallDetails',
                ('method', 'timeout', 'metadata', 'credentials')),
            grpc.ClientCallDetails):
        """
        Implement grpc.ClientCallDetails to pass modified request details in interceptor.
        """
        pass
    
    
    class ArunaClient(object):
        """
         Class to contain the Aruna gRPC client service stubs for easier usage.
        """
        def __init__(self, ):
            ssl_credentials = grpc.ssl_channel_credentials()
            self.secure_channel = grpc.secure_channel("{}:{}".format(ARUNA_HOST, ARUNA_PORT), ssl_credentials)
            self.intercept_channel = grpc.intercept_channel(self.secure_channel, _MyAuthInterceptor())
    
            self.info_client = StorageStatusServiceStub(self.intercept_channel)
            self.auth_client = AuthorizationServiceStub(self.intercept_channel)
            self.user_client = UserServiceStub(self.intercept_channel)
            self.project_client = ProjectServiceStub(self.intercept_channel)
            self.collection_client = CollectionServiceStub(self.intercept_channel)
            self.dataset_client = DatasetServiceStub(self.intercept_channel)
            self.object_client = ObjectServiceStub(self.intercept_channel)
            # self.other_client = ...


    # Entry point of the script
    if __name__ == '__main__':
        # Instantiate ArunaClient
        client = ArunaClient()
        
        # Do something with the client services ...
    ```

=== ":simple-python: Python (simple)"

    To use the Python API library in your Python project you have to install the [PyPI package](https://pypi.org/project/Aruna-Python-API/){:target="_blank"}: `pip install Aruna-Python-API`.

    !!! note
    
        This example does not consider adding the authorization token metadata to every request. 

        In this case you have to manually add the authorization token header by using the `with_call(...)` extension of the client service methods.

        **All Python examples in this documentation assume that the client services have been initialized with an interceptor.**

    ```python linenums="1"
    import grpc

    from aruna.api.storage.services.v2.info_service_pb2_grpc import StorageStatusServiceStub
    from aruna.api.storage.services.v2.authorization_service_pb2_grpc import AuthorizationServiceStub
    from aruna.api.storage.services.v2.user_service_pb2_grpc import UserServiceStub
    from aruna.api.storage.services.v2.project_service_pb2_grpc import ProjectServiceStub
    from aruna.api.storage.services.v2.collection_service_pb2_grpc import CollectionServiceStub
    from aruna.api.storage.services.v2.dataset_service_pb2_grpc import DatasetServiceStub
    from aruna.api.storage.services.v2.object_service_pb2_grpc import ObjectServiceStub
    
    # Valid Aruna API token
    #   In a production environment this should be stored in a more secure location ...
    API_TOKEN = 'MySecretArunaApiToken'
    
    # Aruna instance gRPC endpoint
    ARUNA_HOST = '<URL-To-Aruna-Instance-gRPC-Endpoint>' # Protocol (e.g. https://) has to be omitted
    ARUNA_PORT = '443'
    

    class ArunaClient(object):
        """
        Class to contain the Aruna gRPC client service stubs for easier usage.
        """
        def __init__(self):
            # Read TLS credentials from local trusted certificates and instantiate a channel
            ssl_credentials = grpc.ssl_channel_credentials()
            self.channel    = grpc.secure_channel("{}:{}".format(ARUNA_HOST, ARUNA_PORT), ssl_credentials)

            self.info_client = StorageStatusServiceStub(self.channel)
            self.auth_client = AuthorizationServiceStub(self.channel)
            self.user_client = UserServiceStub(self.channel)
            self.project_client = ProjectServiceStub(self.channel)
            self.collection_client = CollectionServiceStub(self.channel)
            self.dataset_client = DatasetServiceStub(self.channel)
            self.object_client = ObjectServiceStub(self.channel)
            # self.xyz_client = ...


    # Entry point of the Python script
    if __name__ == '__main__':
        # Instantiate ArunaClient
        client = ArunaClient()
        
        # Do something with the client services ... for example:
        response, call = client.user_client.GetUser.with_call(
            request=GetUserRequest(),
            metadata=(('Authorization', f"Bearer {API_TOKEN}"),)
        )
    ```


## User registration

Users can register themselves with an email address and an individual display name in an Aruna instance. 
You only need the valid OIDC token received from one of the supported AAI logins.

The `display_name` and `email` parameters are mandatory while `project` is optional. 
The provided email will only be used for system-relevant notifications e.g. advance notifications of maintenance.
The project parameter is a hint for the administrators to associate the newly registered user with a project for identification purposes.

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # Native JSON request to register OIDC user
    curl -d '
      {
        "display_name": "Forename Surname",
        "email": "forename.surname@example.com", 
        "project": "My little science project"
      }' \ 
         -H 'Authorization: Bearer <AUTH_TOKEN>' \ 
         -H 'Content-Type: application/json' \ 
         -X POST https://<URL-to-Aruna-instance-API-endpoint>/v2/user/register
    ```

=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to register OIDC user
    let request = RegisterUserRequest {
        display_name: "Forename Surname".to_string(),
        email: "forename.surname@example.com".to_string(),
        project: "My little science project".to_string(),
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = user_client.register_user(request)
                              .await
                              .unwrap()
                              .into_inner();
    
    // Do something with the response
    println!("Registered user: {:#?}", response.user_id)
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to register OIDC user
    request = RegisterUserRequest(
        display_name="Forename Surname",
        email="forename.surname@example.com",
        project="My little science project"
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.user_client.RegisterUser(
        request=request
    )
    
    # Do something with the response
    print(f'{response}')
    ```


## User activation

After registration users additionally have to be activated once in a second step to prevent misuse of the system.

??? Abstract "Required permissions"

    Users can only be activated by Aruna global administrators.

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # For convenience, administrators can request info on all unactivated users at once
    curl -H 'Authorization: Bearer <AUTH_TOKEN>' \ 
         -H 'Content-Type: application/json' \ 
         -X GET https://<URL-to-Aruna-instance-API-endpoint>/v2/user/not_activated
    ```

    ```bash linenums="1"
    # Native JSON request to activate registered user
    curl -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X PATCH https://<URL-to-Aruna-instance-API-endpoint>/v2/user/{user-id}/activate
    ```


=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to fetch all not activated users
    let request = GetNotActivatedUsersRequest {};
    
    // Send the request to the Aruna instance gRPC gateway
    let unactivated = user_client.get_not_activated_users(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", unactivated);
    ```

    ```rust linenums="1"
    // Create tonic/ArunaAPI request for user activation    
    let request = ActivateUserRequest {
        user_id: "<user-id>".to_string() // Has to be a valid ULID of a registered user
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = user_client.activate_user(request)
                                       .await
                                       .unwrap()
                                       .into_inner();
    
    // Do something with the response
    println!("Activated user: {:#?}", response.user_id)
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to fetch all not activated users
    request = GetNotActivatedUsersRequest()
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.user_client.GetNotActivatedUsers(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```

    ```python linenums="1"
    # Create tonic/ArunaAPI request for user activation
    request = ActivateUserRequest(
        user_id="<user-id>"  # Has to be a valid ULID of a registered user
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.user_client.ActivateUser(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```


## Who am I / What am I

To check which user a token is associated with or get information about the current users permissions, you can use the UserService API.

??? Abstract "Required permissions"

    Registered users do not need special permissions to fetch information about their user account.

    Only Aruna global administrators can request user information of other users i.e. set the `user_id` parameter of the request to the id of another user.

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # Native JSON request to fetch user information associated with authorization token
    curl -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X GET https://<URL-to-Aruna-instance-API-endpoint>/v2/user
    ```

    ```bash linenums="1"
    # Native JSON request to fetch user information associated with the provided user id
    curl -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X GET 'https://<URL-to-Aruna-instance-API-endpoint>/v2/user?userId=<user-id>'
    ```

=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to fetch user info of current user
    let request = GetUserRequest {
        user_id: "".to_string()
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = user_client.get_user(request)
                              .await
                              .unwrap()
                              .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to fetch user info of current user
    request = GetUserRequest()
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.user_client.GetUser(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```
