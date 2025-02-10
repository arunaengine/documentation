
# How to use the Project API / ProjectServiceClient

## Introduction

Here you get a quick rundown how to basically create, read, update and delete Projects within Aruna.

This should be the first step after gaining access to the storage (and maybe creating an API token) which is described in the previous chapters.

??? Info "What is a Project?"

    A Project is the basic resource to organize general user access for stored data <!--and/or data to be stored--> (i.e. Objects). 

    It also acts as an umbrella container for all other resources which means that every hierarchy has a Project as root. 
    This directly implies that every project name has to be globally unique in the Aruna universe.

    More in-depth information can be found in the [**:material-graph:Data Structure**](../../internal_data_structure/internal_data_structure.md){target="_blank"} section.


## Create Project

API examples of how to create a new Project.
The project creator is automatically granted ADMIN permissions on the created Project.

??? Abstract "Required permissions"

    To create a new Project you only have to be a registered Aruna user.

!!! Info "Project naming guidelines"

    * Project names are unique globally in the Aruna system
    * Project names are restricted to the following characters: [a-z0-9\-] (i.e. alphanumeric lowercase and hyphens)

=== ":simple-curl: cURL"

    :material-checkbox-marked-outline: Tested

    ```bash linenums="1"
    # Native JSON request to create a simple Project
    curl -d '
      {
        "name": "json-api-project", 
        "title": "JSON API Project"
        "description": "Created with JSON over HTTP.",
        "keyValues": [],
        "relations": [],
        "data_class": "DATA_CLASS_PUBLIC",
        "preferredEndpoint": "",
        "metadataLicenseTag": "CC-BY-4.0",
        "defaultDataLicenseTag": "CC-BY-4.0",
        "authors": []
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X POST https://<URL-to-Aruna-instance-API-endpoint>/v2/projects
    ```

=== ":simple-rust: Rust"

    :material-checkbox-blank-outline: Tested

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to create a simple Project
    let request = CreateProjectRequest {
        name: "rust-api-project".to_string(),
        title: "Rust API Project".to_string(),
        description: "Created with the gRPC Rust API client.".to_string(),
        key_values: vec![],
        relations: vec![],
        data_class: DataClass::Public as i32,
        preferred_endpoint: "".to_string(), // Can be set to specific endpoint
        metadata_license_tag: "CC-BY-4.0".to_string(),
        default_data_license_tag: "CC-BY-4.0".to_string(),
        authors: vec![]
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.create_project(request)
                                 .await
                                 .unwrap() //
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to create a simple Project
    request = CreateProjectRequest(
        name="python-api-project",
        title="Python API Project",
        description="Created with the gRPC Python API client.",
        key_values=[], 
        relations=[], 
        data_class=DataClass.DATA_CLASS_PUBLIC,
        preferred_endpoint="",
        metadata_license_tag="CC-BY-4.0",
        default_data_license_tag="CC-BY-4.0",
        authors=[]
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.CreateProject(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```


## Get Project

API example for fetching info of an existing Project.

??? Abstract "Required permissions"

    This request requires at least READ permissions on the specific Project.

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # Native JSON request to fetch information of a Project
    curl -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X GET https://<URL-to-Aruna-instance-API-endpoint>/v2/projects/{project-id}
    ```

=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to fetch information of a Project
    let request = GetProjectRequest { 
        project_id: "<project-id>".to_string(),
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.get_project(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to fetch information of a Project
    request = GetProjectRequest(
        project_id="<project-id>"
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.GetProject(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```


## Get multiple Projects

API examples of how to fetch multiple Projects in a single request.

??? Abstract "Required permissions"

    This request requires at least READ permissions on all requested Projects.

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # Native JSON request to fetch information of multiple projects in one request
    curl -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X GET https://<URL-to-Aruna-instance-API-endpoint>/v2/projects?projectIds=project-id-01&projectIds=project-id-02
    ```

=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to fetch information of multiple projects in one request
    let request = GetProjectsRequest {
        project_ids: vec![
            "<project-id-01>".to_string(),
            "<project-id-02>".to_string(),
            "<...>".to_string(),
        ],
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.get_projects(request)
                              .await
                              .unwrap()
                              .into_inner();
    
    // Do something with the response
    println!("{:#?}\n", response);
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to fetch information of multiple projects in one request
    request = GetProjectsRequest(
        project_ids=["<project-id-01>", "<project-id-02>", "<...>"] 
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.GetProjects(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```


## Update Project

API examples of how to update individual metadata of an existing Project.

??? Abstract "Required permissions"

    * Name update needs at least ADMIN permissions on the specific Project
    * Description update needs at least WRITE permissions on the specific Project
    * KeyValue update needs at least WRITE permissions on the specific Project
    * Dataclass update needs at least ADMIN permissions on the specific Project
    * License update needs at least WRITE permissions on the specific Project

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # Native JSON request to update the name of a Project
    curl -d '
      {
        "name": "updated-json-api-project"
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X PATCH https://<URL-to-Aruna-instance-API-endpoint>/v2/projects/{project-id}/name
    ```

    ```bash linenums="1"
    # Native JSON request to update the title of a Project
    curl -d '
      {
        "title": "Updated JSON API Project"
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X PATCH https://<URL-to-Aruna-instance-API-endpoint>/v2/project/{project-id}/title
    ```

    ```bash linenums="1"
    # Native JSON request to update the description of a Project
    curl -d '
      {
        "description": "Updated with JSON over HTTP."
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X PATCH https://<URL-to-Aruna-instance-API-endpoint>/v2/projects/{project-id}/description
    ```

    ```bash linenums="1"
    # Native JSON request to update the key-values associated with a Project
    curl -d '
      {
        "addKeyValues": [],
        "removeKeyValues": []
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X PATCH https://<URL-to-Aruna-instance-API-endpoint>/v2/projects/{project-id}/key_values
    ```

    !!! Info

        Dataclass can only be relaxed: `Confidential` > `Workspace` > `Private` > `Public`

    ```bash linenums="1"
    # Native JSON request to update the dataclass of a Project
    curl -d '
      {
        "dataClass": "DATA_CLASS_PUBLIC"
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X PATCH https://<URL-to-Aruna-instance-API-endpoint>/v2/projects/{project-id}/data_class
    ```

    ```bash linenums="1"
    # Native JSON request to update the license of a Project
    curl -d '
      {
        "metadataLicenseTag": "CC0",
        "defaultDataLicenseTag": "CC0"
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X PATCH https://<URL-to-Aruna-instance-API-endpoint>/v2/projects/{project-id}/licenses
    ```

    ```bash linenums="1"
    # Native JSON request to add an author to a Project
    curl -d '
      {
        "addAuthors": [
            {
            "firstName": "John",
            "lastName": "Doe",
            "email": "john.doe@example.com",
            "orcid": "0000-0002-1825-0097",
            "id": "<user-id-if-registered>"
            }
        ],
        "removeAuthors": []
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X PATCH https://<URL-to-Aruna-instance-API-endpoint>/v2/project/{project-id}/authors
    ```

=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to update the name of a Project
    let request = UpdateProjectNameRequest {
        project_id: "<project-id>".to_string(),
        name: "updated-rust-api-project".to_string(),
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.update_project_name(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to update the title of a Project
    let request = UpdateProjectTitleRequest {
        project_id: "<project-id>".to_string(),
        title: "Update Rust API Project".to_string(),
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.update_project_title(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to update the description of a Project
    let request = UpdateProjectDescriptionRequest {
        project_id: "<project-id>".to_string(),
        description: "Updated with the gRPC Rust API client.".to_string(),
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.update_project_description(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to update the key-values associated with a Project
    let request = UpdateProjectKeyValuesRequest {
        project_id: "<project-id>".to_string(),
        add_key_values: vec![], 
        remove_key_values: vec![]
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.update_project_key_values(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

    !!! Info

        Dataclass can only be relaxed: `Confidential` > `Private` > `Public`

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to update the datacalass of a Project
    let request = UpdateProjectDataClassRequest {
        project_id: "<project-id>".to_string(),
        data_class: DataClass::Public as i32,
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.update_project_data_class(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to update the licenses of a Project
    let request = UpdateProjectLicensesRequest {
        project_id: "<project-id>".to_string(),
        metadata_license_tag: "CC0".to_string(),
        default_data_license_tag: "CC0".to_string(),
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.update_project_licenses(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to add an author to a Project
    let request = UpdateProjectAuthorsRequest {
        project_id: "<project-id>".to_string(),
        add_authors: vec![Author {
            first_name: "John".to_string(),
            last_name: "Doe".to_string(),
            email: "john.doe@example.com".to_string(),
            orcid: "0000-0002-1825-0097".to_string(),
            id: "<user-id-if-registered>".to_string(),
        }],
        remove_authors: vec![],
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.update_project_authors(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to update the name of a Project
    request = UpdateProjectNameRequest(
        project_id="<project-id>",
        name="updated-python-api-project"
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.UpdateProjectName(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```

    ```python linenums="1"
    # Create tonic/ArunaAPI request to update the title of a Project
    request = UpdateProjectNameRequest(
        project_id="<project-id>",
        title="Updated Python API Project"
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.UpdateProjectTitle(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```

    ```python linenums="1"
    # Create tonic/ArunaAPI request to update the description of a Project
    request = UpdateProjectDescriptionRequest(
        project_id="<project-id>",
        description="Updated with the gRPC Python API client"
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.UpdateProjectDescription(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```

    ```python linenums="1"
    # Create tonic/ArunaAPI request to update the key-values associated with a Project
    request = UpdateProjectKeyValuesRequest(
        project_id="<project-id>",
        add_key_values=[],
        remove_key_values=[]
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.UpdateProjectKeyValues(request=request)
    
    # Do something with the response
    print(f'{response}')
    ``` 

    !!! Info

        Dataclass can only be relaxed: `Confidential` > `Private` > `Public`

    ```python linenums="1"
    # Create tonic/ArunaAPI request to update the data class of a Project
    request = UpdateProjectDataClassRequest(
        project_id="<project-id>",
        data_class=DataClass.DATA_CLASS_PUBLIC
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.UpdateProjectDataClass(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```

    ```python linenums="1"
    # Create tonic/ArunaAPI request to update the licenses of a Project
    request = UpdateProjectLicensesRequest(
        project_id="<project-id>",
        metadata_license_tag="CC0",
        default_data_license_tag="CC0"
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.UpdateProjectLicenses(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```

    ```python linenums="1"
    # Create tonic/ArunaAPI request to add an author to a Project
    request = UpdateProjectAuthorsRequest(
        project_id="<project-id>",
        add_authors=[Author(
            first_name="John",
            last_name="Doe",
            email="john.doe@example.com",
            orcid="0000-0002-1825-0097",
            user_id="<user-id-if-registered"
        )],
        remove_authors=[]
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.UpdateProjectAuthors(request=request)
    
    # Do something with the response
    print(f'{response}')
    ``` 


## Archive Project

A Project can be archived which sets it and all the downstream relations to an immutable read-only state.

??? Abstract "Required permissions"

    This request requires at least ADMIN permissions on the specific Project.

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # Native JSON request to archive a Project
    curl -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X PATCH https://<URL-to-Aruna-instance-API-endpoint>/v2/projects/{project-id}/archive
    ```

=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to archive a Project
    let request = ArchiveProjectRequest {
        project_id: "<project-id>".to_string()
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.archive_project(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to archive a Project
    request = UpdateProjectDescriptionRequest(
        project_id="<project-id>"
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.ArchiveProject(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```


## Delete Project

API examples of how to delete a Project.

!!! Info

    Deletion does not remove the project from the database, but sets the status of the Project and the underlying resources to "DELETED".

??? Abstract "Required permissions"

    This request requires at least ADMIN permissions on the specific Project.

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # Native JSON request to delete a project
    curl -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X DELETE https://<URL-to-Aruna-instance-API-endpoint>/v2/projects/{project-id}
    ```

=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to delete a project
    let request = DeleteProjectRequest {
        project_id: "<project-id>".to_string(),
    };
    
    // Send the request to the Aruna instance gRPC endpoint
    let response = project_client.delete_project(request)
                                 .await
                                 .unwrap()
                                 .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to delete a project
    request = DeleteProjectRequest(
        project_id="<project-id>"
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.project_client.DeleteProject(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```
