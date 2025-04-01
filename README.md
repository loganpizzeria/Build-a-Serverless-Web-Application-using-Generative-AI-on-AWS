# Serverless Web Application using Generative AI on AWS

## Overview
This project is a **serverless web application** that generates recipes using **Generative AI on AWS**. It is built using React for the frontend, AWS Amplify for hosting and authentication, and **Amazon Bedrock** with **Claude 3 Sonnet** for AI-powered recipe generation. The backend is powered by AWS Lambda and GraphQL API, allowing seamless interaction with the AI model.

## Architecture Diagram
The application architecture is outlined in the accompanying **"Diagram_ Serverless Web Application using Generative AI on AWS.pdf"**.

## Features
- **React-based frontend**, hosted on AWS Amplify
- **Continuous deployment** via GitHub integration
- **User authentication** powered by Amazon Cognito
- **AI-generated recipes** using Amazon Bedrock and Claude 3 Sonnet
- **GraphQL API** to process user input and interact with the AI model
- **AWS Lambda function** to construct prompts and handle API requests

## Technologies Used
- **React + TypeScript**
- **AWS Amplify**
- **Amazon Cognito (Amplify Auth)**
- **Amazon Bedrock** (Claude 3 Sonnet Model)
- **GraphQL API**
- **AWS Lambda**
- **AWS IAM Policies**

---

## Setup Instructions

### 1. Create the React Application
```sh
npm create vite@latest ai-recipe-generator -- --template react-ts -y
cd ai-recipe-generator
npm install
npm run dev
```

### 2. Install and Initialize AWS Amplify
```sh
npm create amplify@latest -y
```

### 3. Install Required Packages
```sh
npm install aws-amplify @aws-amplify/ui-react
```

### 4. Set Up Authentication (Cognito)
Update the `amplify/auth/resource.ts` file to customize the verification email:
```ts
import { defineAuth } from "@aws-amplify/backend";

export const auth = defineAuth({
  loginWith: {
    email: {
      verificationEmailStyle: "CODE",
      verificationEmailSubject: "Welcome to the AI-Powered Recipe Generator!",
      verificationEmailBody: (createCode) =>
        `Use this code to confirm your account: ${createCode()}`,
    },
  },
});
```

### 5. Configure Amazon Bedrock as a Data Source
Update `amplify/backend.ts`:
```ts
import { defineBackend } from "@aws-amplify/backend";
import { data } from "./data/resource";
import { PolicyStatement } from "aws-cdk-lib/aws-iam";
import { auth } from "./auth/resource";

const backend = defineBackend({
  auth,
  data,
});

const bedrockDataSource = backend.data.resources.graphqlApi.addHttpDataSource(
  "bedrockDS",
  "https://bedrock-runtime.us-east-1.amazonaws.com",
  {
    authorizationConfig: {
      signingRegion: "us-east-1",
      signingServiceName: "bedrock",
    },
  }
);

bedrockDataSource.grantPrincipal.addToPrincipalPolicy(
  new PolicyStatement({
    resources: [
      "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0",
    ],
    actions: ["bedrock:InvokeModel"],
  })
);
```

### 6. Define GraphQL API Query
Update `ai-recipe-generator/amplify/data/resource.ts`:
```ts
import { type ClientSchema, a, defineData } from "@aws-amplify/backend";

const schema = a.schema({
  BedrockResponse: a.customType({
    body: a.string(),
    error: a.string(),
  }),
  askBedrock: a
    .query()
    .arguments({ ingredients: a.string().array() })
    .returns(a.ref("BedrockResponse"))
    .authorization((allow) => [allow.authenticated()])
    .handler(
      a.handler.custom({ entry: "./bedrock.js", dataSource: "bedrockDS" })
    ),
});

export type Schema = ClientSchema<typeof schema>;

export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: "apiKey",
    apiKeyAuthorizationMode: {
      expiresInDays: 30,
    },
  },
});
```

### 7. Implement Recipe Generation Logic
Update `bedrock.js` to construct the HTTP request and parse AI responses:
```ts
export function request(ctx) {
    const { ingredients = [] } = ctx.args;
    const prompt = `Suggest a recipe idea using these ingredients: ${ingredients.join(", ")}.`;
    return {
      resourcePath: `/model/anthropic.claude-3-sonnet-20240229-v1:0/invoke`,
      method: "POST",
      params: {
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          anthropic_version: "bedrock-2023-05-31",
          max_tokens: 1000,
          messages: [
            { role: "user", content: [{ type: "text", text: `\n\nHuman: ${prompt}\n\nAssistant:` }] },
          ],
        }),
      },
    };
}

export function response(ctx) {
    const parsedBody = JSON.parse(ctx.result.body);
    return { body: parsedBody.content[0].text };
}
```

### 8. Configure the Frontend
Update `src/main.tsx`:
```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";
import "./index.css";
import { Authenticator } from "@aws-amplify/ui-react";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <Authenticator>
      <App />
    </Authenticator>
  </React.StrictMode>
);
```

Update `src/App.tsx`:
```tsx
import { FormEvent, useState } from "react";
import { Amplify } from "aws-amplify";
import { generateClient } from "aws-amplify/data";
import outputs from "../amplify_outputs.json";
import "./App.css";

Amplify.configure(outputs);
const amplifyClient = generateClient({ authMode: "userPool" });

function App() {
  const [result, setResult] = useState("");
  const [loading, setLoading] = useState(false);

  const onSubmit = async (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    setLoading(true);
    try {
      const formData = new FormData(event.currentTarget);
      const { data } = await amplifyClient.queries.askBedrock({
        ingredients: [formData.get("ingredients")?.toString() || ""],
      });
      setResult(data?.body || "No data returned");
    } catch (e) {
      alert(`Error: ${e}`);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <h1>AI Recipe Generator</h1>
      <form onSubmit={onSubmit}>
        <input type="text" name="ingredients" placeholder="Enter ingredients" />
        <button type="submit">Generate</button>
      </form>
      <p>{loading ? "Loading..." : result}</p>
    </div>
  );
}
export default App;
```

