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

### Note the Database Information in Secrets Manager
Get the generated password, TBD

### Test RDS with a SQL Query
Navigate to the query editor and execute a simple query to confirm that the database is working properly.

### Test the AppSync API with a GraphQL Query
Run a simple GraphQL query in the web console to see that the API is functioning properly.

### Test the API Key
Use `curl` or Postman to ping the endpoint with the API key and see that auth is functioning correctly.

## Salesforce Configuration
### Obtain a Salesforce Org and Load Sample Data
Work with your account rep to get Salesforce Connect enabled in a sandbox of your choice. Alternatively, you can [sign up for a Developer Edition org](https://developer.salesforce.com/signup) or get a Salesforce org via our [Trailhead learning platform](https://trailhead.salesforce.com/).

Once the org is created and you can log in, take advantage of the sample data here to see how customer data in Salesforce can be augmented with data from external systems:

- Create a Text field on the Account object called `customerID`, making sure that it's marked as both an **External ID** and also **Unique**.
- Import the data the `sample-customers.csv` as Accounts and Contacts using the [Data Import Wizard](https://trailhead.salesforce.com/en/content/learn/projects/import-and-export-with-data-management-tools/use-the-data-import-wizard), being careful to import the first column into the new `customerID` field.

The sample data here is meant to represent a realistic scenario in which customer data is stored in Salesforce, but Order and Product data is in Amazon RDS. Even though there are disparate data sources, the Order table has a foreign key identifying the customer that placed the order. If this natural key can also be found in Salesforce, it can be used to create an Indirect Lookup that links Orders to Accounts.

When the Orders are surfaced as an External Object, they can be seen as a Related List on the Account page. This provides a convenient view of a customer's recent orders for support agents and sellers working in Salesforce.

### Named Credential in Salesforce
TBD

### External Data Source 
TBD

### Surface the Data in the Salesforce UI
TBD