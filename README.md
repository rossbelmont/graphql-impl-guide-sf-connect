# Implementation Guide: Salesforce Connect adapter for GraphQL
## Introduction and Goals
The purpose of this resource is to guide technical teams administering Salesforce and AWS through the process of configuring and testing the Salesforce Connect adapter for GraphQL against a known, supported configuration. Salesforce recommends customers deploy this simple (yet complete) solution to gain familiarity with all the relevant concepts, then build on that success and extend these patterns to solve for your particular target use case.

The configuration steps detailed below are sequenced so that the each element of the architecture can be validated incrementally, minimizing hassles and troubleshooting. Working step-by-step isolates potential issues, builds the team’s skills in a deliberate manner, and raises overall confidence in the solution. Follow the sequence below to maximize your success.

## Overview of Configuration Steps
Since there are many steps involved, it’s helpful to have a sense for how this will progress before you begin. Here’s an overview of the process:

1. Obtain access to an AWS account suitable for testing and experimentation
2. Create the initial set of AWS resources by running the CloudFormation template
3. Make note of the database name and credentials in Secrets Manager
4. Validate the RDS configuration with a test SQL query
5. Validate AppSync’s connection to the database with a test GraphQL query
6. Validate the AppSync API’s authentication settings with a test HTTP call
7. Obtain access to a Salesforce org suitable for testing and experimentation and load the sample data
8. Define the connectivity and authentication to the GraphQL API endpoint as a Named Credential in Salesforce
9. Create Salesforce’s representation of the External Data Source and create the External Objects via metadata sync
10. Surface the underlying data in the Salesforce UI 
11. Attach your own database table to the GraphQL API and repeat steps 4-10 as needed

These high-level configuration steps are divided into two fundamental groupings: those performed within AWS, and those performed within Salesforce. This allows you to isolate any configuration issues and save time troubleshooting.

> **Note**: do not proceed to the Salesforce configuration (step 7) until the AWS configuration has been validated.

## AWS Configuration

### Obtain an AWS Account
If you don’t already have an AWS account designated for development and testing, use [this short guide](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/) to obtain one.

### Create the AWS Resources via CloudFormation
To save time, the AWS resources used in this example can be created via [this CloudFormation template](https://github.com/aws-samples/aws-appsync-sample-setup/blob/main/AppSyncGraphQLRDS.yaml) hosted in [this GitHub repository](https://github.com/aws-samples/aws-appsync-sample-setup). 

- Log in to your AWS account, then navigate to CloudFormation.
- Choose *Create Stack*
- Under *Prerequisite - Prepare template*, choose *Template is ready*. 
- Under *Template source*, choose *Upload a template file*.
- Upload *AppSyncGraphRDS.yaml* that you downloaded from the previous step.
- Choose Next.
- Provide a name for your stack. For example, GraphQLRDS.
- You can leave the defaults for *Network configuration* or provide a different CIDR ranges.
- Under *Choose a deployment option* choose *Serverless* to create a serverless RDS cluster or *Standard* to create a standard cluster.
- If you chose *Standard* provide options under *RDS Standard Mode Configuration*. You can ignore these settings if you chose *Serverless*.
    - For *RDS Instance Class*, leave the default or choose a larger instance size.
    - For *EC2 instance type*, leave default or choose a larger instance. The EC2 instance (via Cloud9) will allow you to connect to the RDS cluster.
    - For *AutoHibernateTimeout*, leave the default or use a longer window. This option helps save on cost by hibernating your Cloud9 when inactive.
- Provide the following values under *RDS Serverless Mode Configuration* if you chose the *Serverless* option. You can ignore these settings if you chose *Standard* option.
    - Set *AutoPauseCluster* to true if you wish to save cost by scaling down the capacity to 0 when idle. 
    - Provide an appropriate value for *SecondsUntilAutoPause* (if you set *AutoPauseCluster* to true).
    - Set *MinCapacity* to desired minimum capacity of your RDS cluster.
    - Set *MaxCapacity* to desired maximum capacity of your RDS cluster.
- Provide the following values under AppSync
    - For *AppSyncAuthenticationType* choose API_KEY if you want to configure an API KEY for authentication or AWS_IAM for AWS IAM role based authentication.
    - For *GraphQLAdapterLocation* enter *https://github.com/rossbelmont/graphql-impl-guide-sf-connect/blob/main/graphql.resolvers.sql-1.0.0.jar*
    - For *GraphQLAdapterName* enter *graphql.resolvers.sql-1.0.0.jar*
    - For *GraphQLSchemaLocation* enter *https://github.com/rossbelmont/graphql-impl-guide-sf-connect/blob/main/postgres-schema.graphql*
    - For *GraphQLSchemaName* enter *postgres-schema.graphql*
    - For *RDSSchemaLocation* enter *https://github.com/rossbelmont/graphql-impl-guide-sf-connect/blob/main/postgres-ddl.sql*
    - For *RDSSchemaName* enter *postgres-ddl.sql*
- Choose Next, followed by Next.
- Acknowledge by checking the checkbox next to *I acknowledge that AWS CloudFormation might create IAM resources.*
- Choose Submit.


### Note the Database Information in Secrets Manager
- From the AWS console, navigate to Secrets Manager, and choose the secret (it would be either */graphqlrdsserverless/dbsecret* or */graphqlrds/dbsecret*, depending on the configuration.)
- Copy the ARN under *Secret ARN*. 

### Test RDS with a SQL Query
- From the AWS console, navigate to RDS. 
- Choose Query Editor from the left navigation pane.
- Choose your database cluster under *Database instance or cluster*
- Under *Database username* choose *Connect with a Secrets Manager ARN*
- Enter the ARN that you copied from the previous step (*"Note the Database Information in Secrets Manager"*)
- Under *Enter the name of the database*, enter *graphqlrds*
- Run a simple query to confirm that everything is working properly.

### Test the AppSync API with a GraphQL Query
- From the AWS console, navigate to AWS AppSync.
- Choose the GraphQL API that was created by CloudFormation.
- Choose *Queries* from the left navigation pane.
- Run a simple GraphQL query in the Queries console to see that the API is returning the values as expected.

### Test the API Key
Use `curl` or Postman to ping the endpoint with the API key and see that auth is functioning correctly.

## Salesforce Configuration

### Obtain a Salesforce Org and Load Sample Data
Work with your account rep to get Salesforce Connect enabled in a sandbox of your choice. Alternatively, you can [sign up for a Developer Edition org](https://developer.salesforce.com/signup) or get a Salesforce org via our [Trailhead learning platform](https://trailhead.salesforce.com/).

Once the org is created and you can log in, take advantage of the sample data here to see how customer data in Salesforce can be augmented with data from external systems:

- Create a Text field on the Account object called `customerID`, making sure it is marked as both an **External ID** and **Unique**.
- Import the data the `sample-customers.csv` as Accounts and Contacts using the [Data Import Wizard](https://trailhead.salesforce.com/en/content/learn/projects/import-and-export-with-data-management-tools/use-the-data-import-wizard), being careful to import the first column into the new `customerID` field.

The sample data here is meant to represent a realistic scenario in which customer data is stored in Salesforce, but Order and Product data is in Amazon RDS. Even though there are disparate data sources, the Order table has a foreign key identifying the customer who placed the order. If this natural key can also be found in Salesforce, it can be used to create an Indirect Lookup that links Orders to Accounts.

After the AppSync is connected to Salesforce—read on to get to this step—you’ll see that the data in RDS references data imported in this step. The result is that Order data stored in AWS can be seen as a Related List on the Account page.

### Configure the Named Credential
The AWS server resources created in the steps above need to be accessed by Salesforce acting as a client application calling out via HTTPS. This is managed by Salesforce’s Named Credentials capability, which combines the definition of a remote endpoint along with the authentication needed to call that API successfully.

To be more specific, the **External Credential** captures the authentication details, and the **Named Credential** specifies the target endpoint itself. The Named Credential holds a reference to the External Credential so that the callout subsystem knows what authentication to use for the endpoint in question.

#### Access Control via Permission Set
Since Salesforce is secure by default, any User that will use this credential to make a callout needs permission to do so. This is managed via **Permission Sets**, and for convenience, we’ll handle this first:

- Navigate to Setup → **Permission Sets**.
- Identify a Permission Set all the calling Users already have, or click **New** to create a new one specifically for this purpose.
	- This example will use a new Permission Set called **Access External Systems**. Use that as the Label, and press `Tab` to generate an API Name.
- Click **Save**.
- Click **Manage Assignments**, then **Add Assignment** to assign this Permission Set to your User.
- Follow the rest of the wizard to complete the assignment.

#### Authentication: External Credential
You may recall that the AppSync endpoint is protected by an API key. External Credentials capture the authentication configuration, so follow these steps to set up the API key:

- Navigate to Setup → Named Credentials and click the **External Credentials** subtab.
- Click **New** and select **Custom** for the Authentication Protocol.
- Provide a **Label** (API Key Auth for AppSync) and developer **Name** (APIKeyAuthForAppSync), making note of the developer Name you assign for a later step.
- Click **Save**.

![External Credential](https://github.com/rossbelmont/graphql-impl-guide-sf-connect/blob/main/images/ec-edit.png?raw=true)

At this point, the External Credential is created, though we need to store the API key—securely!—and link its use to the Permission Set we defined above. Here’s how that’s accomplished:

- Under **Permission Set Mappings**, click **New**.
- Select the Permission Set we created in the prior step (Access External Systems).
- Under **Authentication Parameters**, click **New**.
- Use `APIKey` as the **Name**, and paste in the API key from AppSync into the **Value** field.
- Click **Save**.

![Permission Set Mapping](https://github.com/rossbelmont/graphql-impl-guide-sf-connect/blob/main/images/perm-set-mapping.png?raw=true)

This stores the API key in an encrypted manner, and links its access to the appropriate permissions. The last step for the External Credential is to configure the HTTP header AppSync expects to contain the API key. Here’s the steps:

- Under **Custom Headers**, click **New**.
- Enter the **Name** of the header required by AppSync: `x-api-key`.
	- This becomes the name of the header in the actual HTTP callout.
- Enter the following formula as the **Value**: `{!$Credential.APIKeyAuthForAppSync.APIKey}`
	- This merge field syntax allows you to reference the API key without reducing security by hard-coding the secret value in clear text.
- Click **Save**.

#### Endpoint: Named Credential
With the permissions and authentication defined, capturing the endpoint as a Named Credential is comparatively simple:

- Navigate to Setup → Named Credentials and click the **Named Credentials** subtab.
- Click **New**
- Enter a descriptive **Label** (AppSync API) and developer **Name** (`AppSyncAPI`), then paste in the **URL** of the API endpoint from AppSync.
- Select the External Credential we created in the prior step (API Key Auth for AppSync)
- Check the checkbox to **Allow Formulas in HTTP Header**.
	- This ensures the formula referencing the API key will be resolved correctly, and not interpreted as literal text.
- Click **Save**.

![Named Credential](https://github.com/rossbelmont/graphql-impl-guide-sf-connect/blob/main/images/nc-edit.png?raw=true)

#### Credential Parameter Storage: User External Credentials
For technical reasons, credential parameters are stored in a standard object called User External Credentials. Users that make callouts with Named Credentials need Read, Create, Edit, and Delete access to this object. Make sure to grant that access via Profiles (shown below) or Permission Sets, even if the User is a System Administrator.

![Granting access to User External Credentials](https://github.com/rossbelmont/graphql-impl-guide-sf-connect/blob/main/images/grant-UCE-Access.gif?raw=true)

### Configure the External Data Source 
The AppSync API acts as an **External Data Source** in Salesforce, which yields one or more **External Objects**. Those need to be configured, though the metadata exposed via AppSync allows you to skip many tedious steps. 

First, create the External Data Source:

- Navigate to Setup → External Data Sources and click **New External Data Source**.
- Enter a descriptive Label for the **External Data Source** (Order Mgmt API) and press `Tab` to generate a developer **Name**.
- Choose GraphQL as the **Type**.
- Select the **Named Credential** we created in the prior step (AppSync API).
- Check the checkbox to allow **Writable External Objects** so that this data can be edited from Salesforce.
- Click **Save**.

Next, we’ll use the exposed metadata to help create the External Objects:

- Click **Validate and Sync**.
- Note the table onscreen with the list of potential objects to add to Salesforce.
	- Tweak the Name and Label fields to match the screenshot below for increased readability.
- Checkmark all three rows with the leftmost column, then click **Sync**.
- Wait for the operation to complete, then note the new External Objects in the list toward the bottom of the screen.

![Sync the External Data Source](https://github.com/rossbelmont/graphql-impl-guide-sf-connect/blob/main/images/sync-xds.png?raw=true)

### Surface the Data in the Salesforce UI
Salesforce has access to the external data at this point, though you’ll want to take a few more steps to surface it to your end users. For the purposes of this test, edit the **Customer ID** field on the new Order object and click Change Field Type to make it an Indirect Lookup to the Account field linked via the **Customer ID** field you added to that standard object.

Once you add the Related List for Orders to the Page Layout for Account, you’ll be able to see the order data from AWS in the context of the customer. This provides a convenient view of a customer's recent orders for support agents and sellers working in Salesforce.

![Order data in Salesforce](https://github.com/rossbelmont/graphql-impl-guide-sf-connect/blob/main/images/orders-under-account.png?raw=true)
