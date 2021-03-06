# The Azure Cosmos DB Workshop

## Overview

Welcome to the Azure Cosmos DB hands-on workshop! Throughout this workshop, you will go through tasks to take a reasonably modern web application, migrate it to [Azure Cosmos DB](https://azure.microsoft.com/services/cosmos-db/), and implement features to make the application better than ever.

After you are done with the workshop, you can continue to use the content available in our [GitHub repository](https://github.com/msusdev/cosmosdb_app_modernization) to learn about Azure Cosmos DB at your own pace.

> [Schedule](./schedule/)

## Goals

In this workshop, you will be asked to:

- Migrate a small dataset from Azure SQL Database to Azure Cosmos DB.
- Write code in Azure Functions to query Azure Cosmos DB.
- Globally distribute your Azure Cosmos DB account to multiple Azure regions.
- Extend the SQL query grammar using a user-defined function (UDF).
- Use Jupyter Notebooks to create a simple dashboard for your database and containers.

## Challenge 1: Deploy the existing application stack

For your first challenge, you will take the existing application stack and deploy it to your Azure subscription.

❗ It may be tempting to rush through this challenge, but taking your time here to learn the environment will help make any future challenges easier to complete.

### Step A: Deploy with an ARM template

The application stack consists of the following components:

- Azure Web App hosting the application website
  - The application code is deployed using the [msusdev/contoso.spaces.web](https://hub.docker.com/r/msusdev/contoso.spaces.web) container.
- Azure Functions to host the back-end APIs that power the application
  - ❗ Any code you modify in this workshop will only happen here.
  - This function app has the [msusdev/contoso.spaces.api.sql](https://hub.docker.com/r/msusdev/contoso.spaces.api.sql) container deployed by default.
- Azure SQL Database and Server for the application data.
- Azure Storage account to host the images used on the website and Azure Functions metadata
- Azure Container Instances to populate the Azure SQL Database and Azure Storage accounts with sample data
  - The [msusdev/contoso.spaces.populate.sql](https://hub.docker.com/r/msusdev/contoso.spaces.populate.sql) container is deployed to the container group.
  - The [msusdev/contoso.spaces.populate.storage](https://hub.docker.com/r/msusdev/contoso.spaces.populate.storage) container is deployed to the container group.

![Existing application architecture](./media/01-arch.png)

Fortunately, you can deploy the application to your Azure subscription using an ARM template:

[![Deploy to Azure](https://docs.microsoft.com/en-us/azure/templates/media/deploy-to-azure.svg)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FMSUSDEV%2Fcosmosdb_app_modernization%2Fmaster%2Farmdeploy.json)

The deployment should take about **10 minutes**.

While the template is deploying, you should check out some of the deployment details from the Azure Resource Manager (ARM) template:

- The username and password for the SQL database are stored as variables in the template: [armdeploy.json#13](https://github.com/MSUSDEV/cosmosdb_app_modernization/blob/fb8685af8a93301801d9f612b35c9dd791de2d79/armdeploy.json#L13)
- The template also contains variables that points to the Docker container images on [Docker Hub](https://hub.docker.com/r/msusdev): [armdeploy.json#15](https://github.com/MSUSDEV/cosmosdb_app_modernization/blob/fb8685af8a93301801d9f612b35c9dd791de2d79/armdeploy.json#L15)
- The template deploys Azure Container Instances to seed the Azure Storage and Azure Cosmos DB accounts with data: [armdeploy.json#91](https://github.com/MSUSDEV/cosmosdb_app_modernization/blob/fb8685af8a93301801d9f612b35c9dd791de2d79/armdeploy.json#L91)
- The Azure Function created by the template has a connection string to the SQL Database: [armdeploy.json#188](https://github.com/MSUSDEV/cosmosdb_app_modernization/blob/fb8685af8a93301801d9f612b35c9dd791de2d79/armdeploy.json#L188)
- The Azure Web App created by the template has multiple configuration settings to point to HTTP APIs hosted on Azure Functions: [armdeploy.json#252](https://github.com/MSUSDEV/cosmosdb_app_modernization/blob/fb8685af8a93301801d9f612b35c9dd791de2d79/armdeploy.json#L252)

### Validation: Review the deployment

1. Browse to the URL for the Azure Web App created by the ARM template deployment. If you have deployed the website successfully, you should see the following homepage:

![Contoso Spaces homepage](./media/01-validation.png)

> [Hints](./hints/01-deploy/)

## Challenge 2: Migrate data from Azure SQL Database to Azure Cosmos DB

For the first "real" challenge in this workshop, we will migrate our existing data from Azure SQL Database to Azure Cosmos DB.

❗ Remember, we are only migrating at this point. We are not changing our code or deleting our existing Azure SQL Database instance.

Since our application is read-only, this opens up a lot of possible avenues to move our data from the existing SQL database to a container in Azure Cosmos DB.

### Step A: Identify the existing database data

The data in Azure SQL Database consists of two simple tables with a predictable relationship:

![Database schema](./media/02-schema.png)

At this point, you should query the Azure SQL Database instance using a tool of your choice and become familiar with the data set.

Next, you should create a map of the data from SQL to JSON formats.

### Step B: Create Azure Cosmos DB resources

It is a best practice to pre-configure your destination resources in Azure Cosmos DB prior to performing your migration.

At this point, you should plan and excute to creation of your Azure Cosmos DB **account**, **database**, and **container**.

### Step C: Perform migration

Now, it's time to perform the migration. There's no need to deal with the complexity of a live/online migration, you can use any tool or manual script you'd like.

### Validation: Plan and execute deployment

1. You should have a plan detailed on how you will perform the migration and how Azure Cosmos DB JSON documents will map to Azure SQL Database records
1. You should have performed the migration. You can verify that your deployment is successful by issuing a query to your new account, database, and container in Azure Cosmos DB. You can use the ``SELECT * FROM locations`` query to test your migration.

> [Hints](./hints/02-migrate/)

> [Solution](./solutions/02-migrate/)

## Challenge 3: Write new code for Azure Functions

In this challenge, you are tasked with replacing the functions within your Azure Function with code that will query Azure Cosmos DB instead.

### Step A: Review existing code

The existing Azure Function that is deployed for your application uses [Entity Framework Core](https://github.com/dotnet/efcore) to [query a SQL database](https://github.com/MSUSDEV/cosmosdb_app_modernization/blob/42abddaa9b3a8f5112a295c88b6092819e475f01/src/Contoso.Spaces.Api.Sql/GetFeaturedLocations.cs#L35).

If you explore the App Service components in the portal, you will quickly notice a few things:

- The Azure Function code is deployed using a Docker container making it read-only
- The Azure Web App has configuration settings for each of the three API it uses

### Step B: Observe the existing APIs

There are three APIs that you will eventually replace:

- ``/api/getfeaturedlocations``: This API returns the four most recently renovated locations
- ``/api/getalllocations``: This API returns all locations
- ``/api/getspecificlocation?id=``: This API returns the location & rooms for a specific location specified by the ``id`` query string parameter

The URL to each of the three APIs is configured in the Azure Web App's configuration settings.

❗ It's highly recommended that you test calling the existing APIs before writing your own so you can become familiar with the way they return results.

### Step C: Write your own code

Now it's time to write your code to replace the outdated SQL code.

❗ You are free to replace the APIs with ones you have written yourself deployed to any service you'd like.

### Validation: Replace API code

1. First, you should delete the existing SQL Database. Confirmation of the database's deletion will be a required step before moving on with the challenges.
1. Your web application should now call the three APIs that query Azure Cosmos DB. This step will be validated by performing a query on Azure Cosmos DB directly using the Data Explorer and validating that the results match the Contoso Spaces website.

> [Hints](./hints/03-refactor/)

> [Solution](./solutions/03-refactor/)

## Challenge 4: Update the SQL query grammar

In this challenge, you will add new functionality to the SQL query grammar that will allow you to manipulate values without changing the underlying data.

The Contoso Spaces company has found out that they need to add a 25% assessment to any office space rentals. This was a recent change made by the local council and it goes into effect in a few days.

### Step A: Update queries

Instead of taking on the arduous tax of changing the data, your team has decided to simply update their queries. The team was to update the query such that:

- Only the **increased monthly rate** that includes the assessment is shown on the pages for each location
- The original unassessed rate, should still be the date stored in the database
- The query should use a programmatic feature of Azure Cosmos DB that stores the assesment factor
- The team would like the ability to change the assessment factor at any time
- The team will not commit to changing the source code for the web application

## Validation: Run updated query

To validate the change was applied successfully, the team will perform the following steps:

- Open the Contoso Spaces website
- Navigate to the "Mcglynn - Kunze Square" location (Location #1)
- Observe the price of the **Pair Corner Office** room.
- This price was originally ``$2,948.00`` but it should now be ``$3,685.00``.

> [Hints](./hints/04-query/)

> [Solution](./solutions/04-query/)

## Challenge 5: Add a Jupyter notebook

In this challenge, you have decided that you'd like to create a single "pane of glass"

### Step A: Create a Jupyter notebook

This is a freeform challenge. You can create any Jupyter notebook using either C# or Python.

The goal is to make a notebook that gathers some data and show it in an interesting way.

Here are some examples of notebooks you can create:

- Notebook that shows the pricing of units and then displays them relative to the latest renovation date
- Notebook that shows the total number of seats per location
- Notebook that shows distribution of locations across different US states

## Validation: View notebook

This challenge is open-ended so you are free to decide on your own validation terms.

> [Hints](./hints/05-notebook/)
