# Composable OAS

This repository provides the example specification for the fintech_devcon 2003 conference presentation "Composable API first design for REST with OpenAPI," which provides guidance for building composable schemas for REST APIs.

## Introduction

When designing APIs organizations that are conscientious of their client side developers' experiences will define standards for their API designs. These standards may address string cases, naming conventions, data type formats, security schemas, etc, and all help improve the usability, security, an maintenance API. Unfortunately, these standards can sometimes be surface level in nature, and are not enough to ensure a quality API.

In short, sometimes an API can be syntactically correct, and still smell. 

Not only does the definition of an API affect it's maintainability and readability, it also affects the client and server side code generation. A poorly structured API definition can affect application performance, and even increase the load that is put on your server. 

The purpose of this repository is to provide guidance for writing API specifications which reduce structural deficiencies between otherwise consistent APIs. It defines a set of resource components to build in your specification that keeps your schema DRY, makes your API schemas consistent, takes advantages of generated validations, and balances the constraints between payload size and server load.

Just as REST is not the answer for every API problem, the guidance in this repository may not be the answer for all your RESTful API needs. However, by identifying the attributes of the properties that are organized into components, and understanding why components are organized as they are, API designers can better define the components that make up the APIs in their domains.

## Schema

This repository contains a schema that exemplifies the guidance provided below.

The schema is for an eCommerce site which takes online orders. The schema is made up of two resources; the `order` and the order `detail`. The fully resolved and simplified schema of these resources would be defined as:

```yaml
components:
  schemas:
    
    detail:
      type: object
      properties:
        id:
          type: string
          format: uuid
        sku:
          type: string
        description:
          type: string
        quantity:
          type: number
    
    order:
      type: object
      properties:
        id:
          type: string
          format: uuid
        status:
          type: string
          enum:
            - placed
            - approved
            - shipped
            - delivered
        isGift:
          type: boolean
        coupon:
          type: string
        details:
          type: array
          items: '#/components/schemas/detail'
        createTime:
          type: string
          format: date-time
        createdBy:
          type: string
        modifyTime:
          type: string
          format: date-time
        modifiedBy:
          type: string
```

Note that the `sku` property refers to the id of a resource that is not defined in the schema. Although it would be considered externally scoped, we will be treating it as a locally scoped property in this example.

The structure of the schema is visualized below:

![Order Schema Visualization](https://github.com/hamilton-dfp/composable-oas/blob/main/resources/audit.png?raw=true)

Please be aware that the schema is for communicating the bare minimum of this pattern, and does not necessarily represent an idealized schema.

## Components

There are two different types of components in this schema, resource components and global components.

### Resource Components

Resource Components are collections of properties for a specific resource that share common attributes. The attributes that are relevent in organizing components are:

* **Owner**: The system responsible for managing the value of the property. This would either be the **server** or the **client**. Server owned properties are those that the client must not have permission to change. This usually includes things like ids and audit fields. Client owned properties are properties that the server could not infer on it's own, such as the name of a resource, what the user wants to order, etc.
  * **Statuses**: Status fields may be tricky to accurately classify here, as in many cases the server does not perform the actions that result in a status change, but at the same time you would not want to let your client have free reign over the value of the status. Typically it is best for a `status` property to be owned by the server, but to be updated based on the actions of the client. For example, when the client makes the order, the server sets the status to `placed`. Once another client creates an approval for the order, the `status` becomes `approved`. If the user creates a cancellation for the order, the `status` becomes `cancelled`, and so forth.
* **Mutability**: Whether the property can be changed (**mutable**) or not (**immutable**), regardless of the owner of the property. Note that sometimes the mutability of a property can change depending on the state of the resource. For example in our `order` schema, we would not allow the `isGift` property to be changed after the order has been shipped, as it is too late to alter the packaging after the item has been shipped. This is not a validation we can perform at the schema level, so this logic needs to be enforced by the server. Therefore, for the sake of our schema, we would consider these properties to be mutable.
* **Scope**: A **locally** scoped property is a property that can be sent from the server to the client with the *least amount of access* to external data sources to represent the resource to a user. For example, if the `order`/`detail` schema above were stored in a relational database, then the `isGift` property would be stored in the `order` table, and would therefore be considered **locally scoped**. The `details` property refers to a sub-resource of the `order`, and would require access to an additional table. Therefore the `details` property would be considered **externally scoped**.

Depending on the domain of the API, there may be other attributes that can affect the organization of resource properties into components, but these attributes will be used for the components defined below.

#### Update component

* Owner: client
* Mutable: true
* Scope: local

The update component is used in `PATCH` requests to update a resource using JSON Merge PATCH. We use JSON Merge PATCH because by doing so we enable this component to be reused as part of other components.

Note that the properties of an update component are locally scoped. If the value of a property is derived from an electronic journal that is external to the resource entity, or is versioned in a seperate data source, you should consider managing the property as a subresource. 

##### Example schema

```yaml
orderUpdate:
  type: object
  properties:
    isGift:
      type: boolean
  minProperties: 1
```

Note that because our `PATCH` request is using JSON Merge Patch, we cannot mark any of these properties as required. However, to generate validation so that we are not processing empty requests, we can use the `minProperties` property to validate that at least one property is set.

##### Example properties
All examples are dependent on the business domain and architecture of the system.

  * Descriptions or user summaries that can be freely changed.
    * Encouter summary, description, treasure fields in D&D Beyond
    * A gift note for an online order.
    * A post on a social media site (assuming edits are not versioned).
  * Quantities
    * The quantity of an order item in an online shopping cart.
    * The rating in an online summary
  * Client owned statuses.
    * The status of a Jira Issue

#### Create component

The create component contains all properties in the update componet as well as properties that meet the following attributes:

* Owner: client
* Mutable: false
* Scope: local

The Create Component is used in `POST` requests to create a resource. The values of the properties are defined by the client, but cannot be changed by the client. All properties must be locally scoped.

##### Example schema

```yaml
orderCreate:
  allOf:
    - $ref: '#/components/schemas/orderUpdate'
    - type: object
      properties:
        coupon:
          type: string
```

Note that in this case, we are not requiring the `coupon` field on creation, as it is easy to have it default to `null`. Also note, however, that by using the `orderUpdate` component that we are adding additional properties to the creation request that are not required. Therefore the server implementation must be prepared to provide default values for any non-required properties.

//TODO it has happened one time that I had a mutable property that could not be defaulted and was therefore required upon creation. I don't recall the use case, only that it happened. I also dont' recall if the issue was specific to the domain, or if it was a fault in the underlying architecture. neither do Iremember how I solved it within this pattern. If you come across such a use case before I am able to update this document, feel free to create an issue so we can figure out how to fit such a use case in this pattern and get it documented.

##### Example properties
All examples are dependent on the business domain and architecture of the system.

* Names
  * While the Id of a resource is how a system identifies a resource, names can be how a human identifies a resource, and therefore some systems will not allow the names of resources to change. For example:
    * Usernames may not change to avoid allowing a sense of anonymity.
    * Product names on an online store not change.
    * Jobs, issues, and other business related listable resources may have an immutable name for identification, but a mutable description that can be updated.
* Classifiers, such as types
  * In an educational program a student account could not be changed into a teacher account, even though they may both use the same account entity in the system.

#### Header component

The header component is named after the Header/Detail UI pattern. The purpose of the header component is to provide the information of a resource that would make it identifiable to a human user while accessing the least amount of external data sources as possible.

The header component properties are organized by the following attributes:

* Owner: client or server
* Mutable: true or false
* Scope: local

The header must always include the resource id, as this will be needed if the client wants to query the full resource itself.

The header component is returned in the response as a list when a `GET /resources` API is called. 

Alternatively, when retrieving a full resource that contains sub-resources to display to a human user, the sub-resources should be represented using the header component. If only the Id of the resource is returned, then the client will need to make additional -- and potentially costly -- requests back to the server for this information anyway.

Depending on the domain and the resource, the information that a user requires to identify a resource may be the locally scoped properties provided during creation (i.e. name, description, type, etc. included in the create component). In such a case the header component should be componsed of the create component. Even if not all properties in the create component are necessary, the benefit of composability is generally worth the extra ding to your payload.

Additionally, audit information such as when the resource was created or modified and by who is generally important for both users and systems to identify a resource.

##### Example schema

```yaml
orderHeader:
  allOf:
    - $ref: '#/components/schemas/audit'
    # optional, include if the create component contains identifying properties. In this 
    # case the `isGift` property may be useful, even though the `coupon` property likely 
    # isn't, so it is being included.
    - $ref: '#/components/schemas/'
    - type: object
      required:
        - id
        - status
      properties:
        id:
          type: string
          format: uuid
        status:
          $ref: '#/components/schemas/orderStatus'
```

##### Example properties

* The purpose of a header component is not always identification, sometimes it is summary. As an example consider the fields that are shown in Jira backlog or sprint board vs. what is displayed in the actual issue.
* User information
  * While a user profile may have a bio, location, contact details, etc. a social media site might user a user header on comments or posts that only contains the user name, id, and avatar.
* Searching
  * Any properties you would use to search or filter resources in a search screen should be part of the resource header, so long as the properties are locally scoped. Any properties that would would like to search or filter on that are not locally scoped might be considered for an advanced search. In such a case it should be communicated that an advanced search may take longer to perform.

#### Resource component

* Owner: client or server
* Mutability: true or false
* Scope: local or external

The resource component is the composition of all other components to provide the most complete view of the resource and any of it's sub-resources. It contains all additional information related to the resource that is not included in the header.

##### Example schema

```yaml
order:
  allOf:
    - $ref: '#/components/schemas/orderHeader'
    - type: object
      required:
        - details
      properties:
      details:
        type: array
        items:
          $ref: '#/components/schemas/detailHeader'
```

##### Example properties

The primary properties that are included in this component that are not in any of the others are externally scoped properties. This is because when creating a resource, the API only requires the ids of any sub-resources, but during retrieval it should display resource headers. 
