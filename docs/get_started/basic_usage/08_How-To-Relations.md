
# How to use the Relation API / RelationsServiceClient

All resources and their relationships form a directed acyclic graph (DAG) with Projects as roots and Objects as leaves. Collections and Datasets can exist directly beneath Projects but only a Dataset and/or Objects can be created inside a Collection.

In our model, we also distinguish internal relations between Aruna resources and external relations which point to resources outside of Aruna e.g. a DOI.

For a deeper dive into the relations concept of Aruna read the [**:material-graph: Data Structure**](../../internal_data_structure/internal_data_structure.md){target="_blank"} section.

## Modify relations

When resources are created, updated and cloned, internal relationships are automatically created between the resources concerned. 
But these relationships can also be modified again without any problems. 
Only with the predefined relationship types must certain rules be adhered to. 
With freely defined relationship types, resources can be linked to each other as desired.

??? Abstract "Required permissions"

    This request requires at least WRITE permissions on all affected resources.

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # Native JSON request to add an Object to another Project
    curl -d '
      {
        "resourceId": "<project-id>",
        "addRelations": [
            {
            "internal": {
                "resourceId": "<object-id>",
                "resourceVariant": "RESOURCE_VARIANT_OBJECT",
                "definedVariant": "INTERNAL_RELATION_VARIANT_BELONGS_TO",
                "customVariant": "",
                "direction": "RELATION_DIRECTION_OUTBOUND"
            }
            }
        ],
        "removeRelations": []
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X POST https://<URL-to-Aruna-instance-API-endpoint>/v2/relations
    ```

    ```bash linenums="1"
    # Native JSON request to move a Collection to another Project
    curl -d '
      {
        "resourceId": "<collection-id>",
        "addRelations": [
          {
            "internal": {
              "resourceId": "<another-project-id>",
              "resourceVariant": "RESOURCE_VARIANT_PROJECT",
              "definedVariant": "INTERNAL_RELATION_VARIANT_BELONGS_TO",
              "customVariant": "",
              "direction": "RELATION_DIRECTION_INBOUND"
            }
          }
        ],
        "removeRelations": [
          {
            "internal": {
              "resourceId": "<original-project-id>",
              "resourceVariant": "RESOURCE_VARIANT_PROJECT",
              "definedVariant": "INTERNAL_RELATION_VARIANT_BELONGS_TO",
              "customVariant": "",
              "direction": "RELATION_DIRECTION_INBOUND"
            }
          }
        ]
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X POST https://<URL-to-Aruna-instance-API-endpoint>/v2/relations
    ```

    ```bash linenums="1"
    # Native JSON request to add an external relation to an Object
    curl -d '
      {
        "resourceId": "<object-id>",
        "addRelations": [
          {
            "external": {
              "identifier": "https://dev.aruna-engine.org/objects/<object-id>",
              "definedVariant": "EXTERNAL_RELATION_VARIANT_URL",
              "customVariant": ""
            },
          }
        ],
        "removeRelations": []
      }' \
         -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X POST https://<URL-to-Aruna-instance-API-endpoint>/v2/relations
    ```

=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to add an Object to another Project
    let request = ModifyRelationsRequest {
        resource_id: "<project-id>".to_string(),
        add_relations: vec![Relation {
            relation: Some(relation::Relation::Internal(InternalRelation {
                resource_id: "<object-id>".to_string(),
                resource_variant: ResourceVariant::Object as i32,
                defined_variant: InternalRelationVariant::BelongsTo as i32,
                custom_variant: None,
                direction: RelationDirection::Outbound as i32,
            })),
        }],
        remove_relations: vec![],
    };

    // Send the request to the Aruna instance gRPC endpoint
    let response = relation_client.modify_relations(request)
                                  .await
                                  .unwrap()
                                  .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to move a Collection to another Project
    let request = ModifyRelationsRequest {
        resource_id: "<collection-id>".to_string(),
        add_relations: vec![Relation {
            relation: Some(relation::Relation::Internal(InternalRelation {
                resource_id: "<another-project-id>".to_string(),
                resource_variant: ResourceVariant::Project as i32,
                defined_variant: InternalRelationVariant::BelongsTo as i32,
                custom_variant: None,
                direction: RelationDirection::Inbound as i32,
            })),
        }],
        remove_relations: vec![
            Relation {
            relation: Some(relation::Relation::Internal(InternalRelation {
                resource_id: "<original-project-id>".to_string(),
                resource_variant: ResourceVariant::Project as i32,
                defined_variant: InternalRelationVariant::BelongsTo as i32,
                custom_variant: None,
                direction: RelationDirection::Inbound as i32,
            })),
        }
        ],
    };

    // Send the request to the Aruna instance gRPC endpoint
    let response = relation_client.modify_relations(request)
                                  .await
                                  .unwrap()
                                  .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to add an external relation to an Object
    let request = ModifyRelationsRequest {
        resource_id: "<project-id>".to_string(),
        add_relations: vec![Relation {
            relation: Some(relation::Relation::External(
                             ExternalRelation {
                               identifier: "https://dev.aruna-engine.org/objects/<object-id>".to_string(),
                               defined_variant: ExternalRelationVariant::Url as i32,
                               custom_variant: None,
                             }
                           )),
        }],
        remove_relations: vec![],
    };

    // Send the request to the Aruna instance gRPC endpoint
    let response = relation_client.modify_relations(request)
                                  .await
                                  .unwrap()
                                  .into_inner();
    
    // Do something with the response
    println!("{:#?}", response);
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to add an Object to another Project
    request = ModifyRelationsRequest(
        resource_id="<project-id>",
        add_relations=[
            Relation(
                internal=InternalRelation(
                    resource_id="<object-id>",
                    resource_variant=ResourceVariant.RESOURCE_VARIANT_OBJECT,
                    defined_variant=InternalRelationVariant.INTERNAL_RELATION_VARIANT_BELONGS_TO,
                    custom_variant=None,
                    direction=RelationDirection.RELATION_DIRECTION_OUTBOUND
                )
            )
        ],
        remove_relations=[]
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.relation_client.ModifyRelations(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```

    ```python linenums="1"
    # Create tonic/ArunaAPI request to move a Collection to another Project
    request = ModifyRelationsRequest(
        resource_id="<collection-id>",
        add_relations=[
            Relation(
                internal=InternalRelation(
                    resource_id="<another-project-id>",
                    resource_variant=ResourceVariant.RESOURCE_VARIANT_PROJECT,
                    defined_variant=InternalRelationVariant.INTERNAL_RELATION_VARIANT_BELONGS_TO,
                    custom_variant=None,
                    direction=RelationDirection.RELATION_DIRECTION_INBOUND
                )
            )
        ],
        remove_relations=[
            Relation(
                internal=InternalRelation(
                    resource_id="<original-project-id>",
                    resource_variant=ResourceVariant.RESOURCE_VARIANT_PROJECT,
                    defined_variant=InternalRelationVariant.INTERNAL_RELATION_VARIANT_BELONGS_TO,
                    custom_variant=None,
                    direction=RelationDirection.RELATION_DIRECTION_INBOUND
                )
            )
        ]
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.relation_client.ModifyRelations(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```

    ```python linenums="1"
    # Create tonic/ArunaAPI request to add an external relation to an Object
    request = ModifyRelationsRequest(
        resource_id="<project-id>",
        add_relations=[
            Relation(
                internal=ExternalRelation(
                    identifier: "https://dev.aruna-engine.org/objects/<object-id>".to_string(),
                    defined_variant: ExternalRelationVariant.EXTERNAL_RELATION_VARIANT_URL,
                    custom_variant: None,
                )
            )
        ],
        remove_relations=[]
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.relation_client.ModifyRelations(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```


## Get resource hierarchy

API examples of how to fetch all downstream hierarchies of a specific resource.

=== ":simple-curl: cURL"

    ```bash linenums="1"
    # Native JSON request to fetch the downstream hierarchies of a specific resource
    curl -H 'Authorization: Bearer <AUTH_TOKEN>' \
         -H 'Content-Type: application/json' \
         -X GET 'https://<URL-to-Aruna-instance-API-endpoint>/v2/relations/{resource-id}/hierarchy'
    ```

=== ":simple-rust: Rust"

    ```rust linenums="1"
    // Create tonic/ArunaAPI request to fetch the downstream hierarchies of a specific resource
    let request = GetHierarchyRequest {
        resource_id: "<resource-id>".to_string(),
    };

    // Send the request to the Aruna instance gRPC endpoint
    let response = relation_client.get_hierarchy(request)
                                  .await
                                  .unwrap()
                                  .into_inner();

    // Do something with the response
    println!("{:#?}", response);
    ```

=== ":simple-python: Python"

    ```python linenums="1"
    # Create tonic/ArunaAPI request to create a Project
    request = GetHierarchyRequest(
        resource_id="<resource-id>"
    )
    
    # Send the request to the Aruna instance gRPC endpoint
    response = client.relation_client.GetHierarchy(request=request)
    
    # Do something with the response
    print(f'{response}')
    ```
