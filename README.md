# Build-a-CRUD-API-with-Lambda-and-DynamoDB

Build a CRUD API with Lambda and DynamoDB

In this tutorial, you create a serverless API that creates, reads, updates, and deletes items from a DynamoDB table. DynamoDB is a fully managed NoSQL database service that provides fast and predictable performance with seamless scalability. This tutorial takes approximately 30 minutes to complete, and you can do it within the AWS Free Tier.

First, you create a DynamoDB table using the DynamoDB console, hen you create a Lambda function using the AWS Lambda console. Next, you create an HTTP API using the API Gateway console. Lastly, you test your API.

When you invoke your HTTP API, API Gateway routes the request to your Lambda function. The Lambda function interacts with DynamoDB, and returns a response to API Gateway. API Gateway then returns a response to you.

    
To complete this exercise, you need an AWS account and an AWS Identity and Access Management user with console access. 

Procedure

Step 1: Create a DynamoDB table

Step 2: Create a Lambda function

Step 3: Create an HTTP API

Step 4: Create routes

Step 5: Create an integration

Step 6: Attach your integration to routes

Step 7: Test your API

Step 8: Clean up


Step 1: Create a DynamoDB table

You use a DynamoDB table to store data for your API.

Each item has a unique ID, which we use as the partition key for the table.

To create a DynamoDB table

Open the DynamoDB console at https://console.aws.amazon.com/dynamodb/.

Choose Create table.

For Table name, enter httpcrud-db

For Partition key, enter id.

Choose Create table.

<img width="919" alt="Screenshot 2023-08-09 at 21 30 11" src="https://github.com/Mamiololo01/Build-a-CRUD-API-with-Lambda-and-DynamoDB/assets/67044030/e3875b01-8ea1-40cf-b58e-4ce8a1be2c3e">

<img width="866" alt="Screenshot 2023-08-09 at 21 30 22" src="https://github.com/Mamiololo01/Build-a-CRUD-API-with-Lambda-and-DynamoDB/assets/67044030/79f861f6-5b4c-4d7f-b52e-fe11f37a685f">

<img width="929" alt="Screenshot 2023-08-09 at 21 31 53" src="https://github.com/Mamiololo01/Build-a-CRUD-API-with-Lambda-and-DynamoDB/assets/67044030/b409c5d1-efee-4038-bf13-49e93e426d59">


Step 2: Create a Lambda function

You create a Lambda function for the backend of your API. This Lambda function creates, reads, updates, and deletes items from DynamoDB. The function uses events from API Gateway to determine how to interact with DynamoDB. For simplicity this tutorial uses a single Lambda function. As a best practice, you should create separate functions for each route.

To create a Lambda function

Sign in to the Lambda console at https://console.aws.amazon.com/lambda.

Choose Create function.

For Function name, enter httpcrudlambda

Under Permissions choose Change default execution role.

Select Create a new role from AWS policy templates.

For Role name, enter httpcrudlambdarole

For Policy templates, choose Simple microservice permissions. This policy grants the Lambda function permission to interact with DynamoDB.

This tutorial uses a managed policy for simplicity. As a best practice, you should create your own IAM policy to grant the minimum permissions required.

Choose Create function.

<img width="1108" alt="Screenshot 2023-08-09 at 21 40 12" src="https://github.com/Mamiololo01/Build-a-CRUD-API-with-Lambda-and-DynamoDB/assets/67044030/f2089baa-57c9-4b7c-97cf-200825fb3ee2">

<img width="1132" alt="Screenshot 2023-08-09 at 21 40 42" src="https://github.com/Mamiololo01/Build-a-CRUD-API-with-Lambda-and-DynamoDB/assets/67044030/fee6599d-0bab-4a0e-8ee5-47dcb42d7d0f">

<img width="1182" alt="Screenshot 2023-08-09 at 21 41 13" src="https://github.com/Mamiololo01/Build-a-CRUD-API-with-Lambda-and-DynamoDB/assets/67044030/907a3d42-b983-403f-ac4a-28a60c318e6c">


Open index.mjs in the console's code editor, and replace its contents with the following code. Choose Deploy to update your function.


import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  ScanCommand,
  PutCommand,
  GetCommand,
  DeleteCommand,
} from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});

const dynamo = DynamoDBDocumentClient.from(client);

const tableName = "http-crud-tutorial-items";

export const handler = async (event, context) => {
  let body;
  let statusCode = 200;
  const headers = {
    "Content-Type": "application/json",
  };

  try {
    switch (event.routeKey) {
      case "DELETE /items/{id}":
        await dynamo.send(
          new DeleteCommand({
            TableName: tableName,
            Key: {
              id: event.pathParameters.id,
            },
          })
        );
        body = `Deleted item ${event.pathParameters.id}`;
        break;
      case "GET /items/{id}":
        body = await dynamo.send(
          new GetCommand({
            TableName: tableName,
            Key: {
              id: event.pathParameters.id,
            },
          })
        );
        body = body.Item;
        break;
      case "GET /items":
        body = await dynamo.send(
          new ScanCommand({ TableName: tableName })
        );
        body = body.Items;
        break;
      case "PUT /items":
        let requestJSON = JSON.parse(event.body);
        await dynamo.send(
          new PutCommand({
            TableName: tableName,
            Item: {
              id: requestJSON.id,
              price: requestJSON.price,
              name: requestJSON.name,
            },
          })
        );
        body = `Put item ${requestJSON.id}`;
        break;
      default:
        throw new Error(`Unsupported route: "${event.routeKey}"`);
    }
  } catch (err) {
    statusCode = 400;
    body = err.message;
  } finally {
    body = JSON.stringify(body);
  }

  return {
    statusCode,
    body,
    headers,
  };
};
Step 3: Create an HTTP API
The HTTP API provides an HTTP endpoint for your Lambda function. In this step, you create an empty API. In the following steps, you configure routes and integrations to connect your API and your Lambda function.

To create an HTTP API
Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

Choose Create API, and then for HTTP API, choose Build.

For API name, enter http-crud-tutorial-api.

Choose Next.

For Configure routes, choose Next to skip route creation. You create routes later.

Review the stage that API Gateway creates for you, and then choose Next.

Choose Create.

Step 4: Create routes
Routes are a way to send incoming API requests to backend resources. Routes consist of two parts: an HTTP method and a resource path, for example, GET /items. For this example API, we create four routes:

GET /items/{id}

GET /items

PUT /items

DELETE /items/{id}

To create routes
Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

Choose your API.

Choose Routes.

Choose Create.

For Method, choose GET.

For the path, enter /items/{id}. The {id} at the end of the path is a path parameter that API Gateway retrieves from the request path when a client makes a request.

Choose Create.

Repeat steps 4-7 for GET /items, DELETE /items/{id}, and PUT /items.


        Your API has routes for GET /items, GET /items/{id},DELETE
            /items/{id}, and PUT /items.
      
Step 5: Create an integration
You create an integration to connect a route to backend resources. For this example API, you create one Lambda integration that you use for all routes.

To create an integration
Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

Choose your API.

Choose Integrations.

Choose Manage integrations and then choose Create.

Skip Attach this integration to a route. You complete that in a later step.

For Integration type, choose Lambda function.

For Lambda function, enter http-crud-tutorial-function.

Choose Create.

Step 6: Attach your integration to routes
For this example API, you use the same Lambda integration for all routes. After you attach the integration to all of the API's routes, your Lambda function is invoked when a client calls any of your routes.

To attach integrations to routes
Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

Choose your API.

Choose Integrations.

Choose a route.

Under Choose an existing integration, choose http-crud-tutorial-function.

Choose Attach integration.

Repeat steps 4-6 for all routes.

All routes show that an AWS Lambda integration is attached.


        The console shows AWS Lambda on all routes to indicate that your integration is attached.
      
Now that you have an HTTP API with routes and integrations, you can test your API.

Step 7: Test your API
To make sure that your API is working, you use curl.

To get the URL to invoke your API
Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

Choose your API.

Note your API's invoke URL. It appears under Invoke URL on the Details page.


            After you create your API, the console shows your API's invoke URL.
          
Copy your API's invoke URL.

The full URL looks like https://abcdef123.execute-api.us-west-2.amazonaws.com.

To create or update an item
Use the following command to create or update an item. The command includes a request body with the item's ID, price, and name.


curl -X "PUT" -H "Content-Type: application/json" -d "{\"id\": \"123\", \"price\": 12345, \"name\": \"myitem\"}" https://abcdef123.execute-api.us-west-2.amazonaws.com/items
To get all items
Use the following command to list all items.


curl https://abcdef123.execute-api.us-west-2.amazonaws.com/items
To get an item
Use the following command to get an item by its ID.


curl https://abcdef123.execute-api.us-west-2.amazonaws.com/items/123
To delete an item
Use the following command to delete an item.


curl -X "DELETE" https://abcdef123.execute-api.us-west-2.amazonaws.com/items/123
Get all items to verify that the item was deleted.


curl https://abcdef123.execute-api.us-west-2.amazonaws.com/items
Step 8: Clean up
To prevent unnecessary costs, delete the resources that you created as part of this getting started exercise. The following steps delete your HTTP API, your Lambda function, and associated resources.

To delete a DynamoDB table
Open the DynamoDB console at https://console.aws.amazon.com/dynamodb/.

Select your table.

Choose Delete table.

Confirm your choice, and choose Delete.

To delete an HTTP API
Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

On the APIs page, select an API. Choose Actions, and then choose Delete.

Choose Delete.

To delete a Lambda function
Sign in to the Lambda console at https://console.aws.amazon.com/lambda.

On the Functions page, select a function. Choose Actions, and then choose Delete.

Choose Delete.

To delete a Lambda function's log group
In the Amazon CloudWatch console, open the Log groups page.

On the Log groups page, select the function's log group (/aws/lambda/http-crud-tutorial-function). Choose Actions, and then choose Delete log group.

Choose Delete.

To delete a Lambda function's execution role
In the AWS Identity and Access Management console, open the Roles page.

Select the function's role, for example, http-crud-tutorial-role.

Choose Delete role.

Choose Yes, delete.

