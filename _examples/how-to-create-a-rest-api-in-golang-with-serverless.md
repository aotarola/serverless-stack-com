---
layout: example
title: How to create a REST API in Golang with serverless
short_title: Go REST API
date: 2021-04-04 00:00:00
lang: en
index: 4
type: api
description: In this example we will look at how to create a serverless REST API on AWS with Golang using SST. We'll be using the Api construct to define the routes of our API.
short_desc: Building a REST API with Golang.
repo: rest-api-go
ref: how-to-create-a-rest-api-in-golang-with-serverless
comments_id: how-to-create-a-rest-api-in-golang-with-serverless/2367
---

In this example we'll look at how to create a serverless REST API with Golang on AWS using [SST]({{ site.sst_github_repo }}).

## Requirements

- Node.js 16 or later for our CDK code
- Golang 1.16 or similar for our Lambda code
- An [AWS account]({% link _chapters/create-an-aws-account.md %}) with the [AWS CLI configured locally]({% link _chapters/configure-the-aws-cli.md %})

## Create an SST app

{%change%} Let's start by creating an SST app.

```bash
$ npx create-sst@latest --template=other/go rest-api-go
$ cd rest-api-go
$ npm install
```

By default, our app will be deployed to the `us-east-1` AWS region. This can be changed in the `sst.config.ts` in your project root.

```js
import { SSTConfig } from "sst";

export default {
  config(_input) {
    return {
      name: "rest-api-go",
      region: "us-east-1",
    };
  },
} satisfies SSTConfig;
```

## Setting up our routes

Let's start by setting up the routes for our API.

{%change%} Add the following below the `config` function in the `sst.config.ts`.

```ts
stacks(app) {
	app.setDefaultFunctionProps({
		runtime: "go1.x",
	});
	app.stack(function Stack({ stack }) {
		const api = new Api(stack, "api", {
			routes: {
				"GET /notes": "functions/lambda/list.go",
				"GET /notes/{id}": "functions/lambda/get.go",
				"PUT /notes/{id}": "functions/lambda/update.go",
			},
		});
		stack.addOutputs({
			ApiEndpoint: api.url,
		});
	});
},
```

We are creating an API here using the [`Api`]({{ site.docs_url }}/constructs/api) construct. And we are adding three routes to it.

```
GET /notes
GET /notes/{id}
PUT /notes/{id}
```

The first is getting a list of notes. The second is getting a specific note given an id. And the third is updating a note.

## Adding function code

For this example, we are not using a database. We'll look at that in detail in another example. So internally we are just going to get the list of notes from a file.

{%change%} Let's add a file that contains our notes in `db/notes.go`.

```go
package db

import (
	"strconv"
	"time"
)

func Notes() map[string]map[string]string {
	return map[string]map[string]string{
		"id1": {
			"noteId":    "id1",
			"userId":    "user1",
			"content":   "Hello World!",
			"createdAt": strconv.FormatInt(time.Now().Unix(), 10),
		},
		"id2": {
			"noteId":    "id2",
			"userId":    "user2",
			"content":   "Hello Old World!",
			"createdAt": strconv.FormatInt(time.Now().Unix()-1000, 10),
		},
	}
}
```

Now add the code for our first endpoint.

### Getting a list of notes

{%change%} Add a `functions/lambda/list.go`.

```go
package main

import (
	"encoding/json"
	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	"github.com/serverless-stack/examples/rest-api-go/db"
)

func Handler(request events.APIGatewayV2HTTPRequest) (events.APIGatewayProxyResponse, error) {
	response, _ := json.Marshal(db.Notes())

	return events.APIGatewayProxyResponse{
		Body:       string(response),
		StatusCode: 200,
	}, nil
}

func main() {
	lambda.Start(Handler)
}
```

Here we are simply converting a list of notes to string, and responding with that in the request body.

### Getting a specific note

{%change%} Add the following to `functions/lambda/get.go`.

```go
package main

import (
	"encoding/json"
	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	"github.com/serverless-stack/examples/rest-api-go/db"
)

func Handler(request events.APIGatewayV2HTTPRequest) (events.APIGatewayProxyResponse, error) {
	var notes = db.Notes()
	var note = notes[request.PathParameters["id"]]

	if note == nil {
		return events.APIGatewayProxyResponse{
			Body:       `{"error":true}`,
			StatusCode: 404,
		}, nil
	}

	response, _ := json.Marshal(note)

	return events.APIGatewayProxyResponse{
		Body:       string(response),
		StatusCode: 200,
	}, nil
}

func main() {
	lambda.Start(Handler)
}
```

Here we are checking if we have the requested note. If we do, we respond with it. If we don't, then we respond with a 404 error.

### Updating a note

{%change%} Add the following to `functions/lambda/update.go`.

```go
package main

import (
	"encoding/json"
	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	"github.com/serverless-stack/examples/rest-api-go/db"
)

func Handler(request events.APIGatewayV2HTTPRequest) (events.APIGatewayProxyResponse, error) {
	var notes = db.Notes()
	var note = notes[request.PathParameters["id"]]

	if note == nil {
		return events.APIGatewayProxyResponse{
			Body:       `{"error":true}`,
			StatusCode: 404,
		}, nil
	}

	var body map[string]string
	_ = json.Unmarshal([]byte(request.Body), &body)

	note["content"] = body["content"]

	response, _ := json.Marshal(note)

	return events.APIGatewayProxyResponse{
		Body:       string(response),
		StatusCode: 200,
	}, nil
}

func main() {
	lambda.Start(Handler)
}
```

We first check if the note with the requested id exists. And then we update the content of the note and return it. Of course, we aren't really saving our changes because we don't have a database!

Now let's test our new API.

## Starting your dev environment

{%change%} SST features a [Live Lambda Development]({{ site.docs_url }}/live-lambda-development) environment that allows you to work on your serverless apps live.

```bash
$ npm run dev
```

The first time you run this command it'll take a couple of minutes to deploy your app and a debug stack to power the Live Lambda Development environment.

```
===============
 Deploying app
===============

Preparing your SST app
Transpiling source
Linting source
Deploying stacks
dev-rest-api-go-ExampleStack: deploying...

 ✅  dev-rest-api-go-ExampleStack


Stack dev-rest-api-go-ExampleStack
  Status: deployed
  Outputs:
    ApiEndpoint: https://rxk5buowgi.execute-api.us-east-1.amazonaws.com
```

The `ApiEndpoint` is the API we just created.

Let's test our endpoint using the integrated [SST Console](https://console.sst.dev). The SST Console is a web based dashboard to manage your SST apps [Learn more about it in our docs]({{ site.docs_url }}/console).

Go to the **API** explorer and click the **Send** button of the `GET /notes` route to get a list of notes.

Note, The [API explorer]({{ site.docs_url }}/console#api) lets you make HTTP requests to any of the routes in your `Api` construct. Set the headers, query params, request body, and view the function logs with the response.

![API tab get notes response](/assets/examples/rest-api/api-tab-get-notes-response.png)

You should see the list of notes as a JSON string.

To retrieve a specific note, Go to `GET /notes/{id}` route and in the **URL** tab enter the **id** of the note you want to get in the **id** field and click the **Send** button to get that note.

![API tab get specific note response](/assets/examples/rest-api/api-tab-get-specific-note-response.png)

Now to update our note, we need to make a `PUT` request, go to `PUT /notes/{id}` route.

In the **URL** tab, enter the **id** of the note you want to update and in the **body** tab and enter the below json value and hit **Send**.

```json
{ "content": "Updating my note" }
```

![API tab update note response](/assets/examples/rest-api/api-tab-update-note-response.png)

This should respond with the updated note.

## Making changes

Let's make a quick change to our API. It would be good if the JSON strings are pretty printed to make them more readable.

{%change%} Replace `Handler` function in `functions/lambda/list.go` with the following.

```go
func Handler(request events.APIGatewayV2HTTPRequest) (events.APIGatewayProxyResponse, error) {
	response, _ := json.MarshalIndent(db.Notes(), "", "  ")

	return events.APIGatewayProxyResponse{
		Body:       string(response),
		StatusCode: 200,
	}, nil
}
```

Here we are just adding some spaces to pretty print the JSON.

If you head back to the `GET /notes` route and hit **Send** again.

![API tab get notes response with spaces](/assets/examples/rest-api/api-tab-get-notes-response-with-spaces.png)
You should see your list of notes in a more readable format.

## Deploying your API

{%change%} To wrap things up we'll deploy our app to prod.

```bash
$ npx sst deploy --stage prod
```

This allows us to separate our environments, so when we are working in `dev`, it doesn't break the app for our users.

Once deployed, you should see something like this.

```bash
 ✅  prod-rest-api-go-ExampleStack


Stack prod-rest-api-go-ExampleStack
  Status: deployed
  Outputs:
    ApiEndpoint: https://ck198mfop1.execute-api.us-east-1.amazonaws.com
```

Run the below command to open the SST Console in **prod** stage to test the production endpoint.

```bash
npx sst console --stage prod
```

Go to the **API** explorer and click **Send** button of the `GET /notes` route, to send a `GET` request.

![Prod API explorer get notes response with spaces](/assets/examples/rest-api/prod-api-tab-get-notes-response-with-spaces.png)

## Cleaning up

Finally, you can remove the resources created in this example using the following commands.

```bash
$ npx sst remove
$ npx sst remove --stage prod
```

## Conclusion

And that's it! You've got a brand new serverless API. A local development environment, to test and make changes. And it's deployed to production as well, so you can share it with your users. Check out the repo below for the code we used in this example. And leave a comment if you have any questions!
