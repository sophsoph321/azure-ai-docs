---
manager: nitinme
author: aahill
ms.author: aahi
ms.service: azure-ai-agent-service
ms.topic: include
ms.date: 02/03/2025
ms.custom: devx-track-js
---


| [Reference documentation](/javascript/api/overview/azure/ai-projects-readme) | [Samples](https://github.com/Azure/azure-sdk-for-js/blob/main/sdk/ai/ai-projects/README.md) | [Library source code](https://github.com/Azure/azure-sdk-for-js/tree/main/sdk/ai/ai-projects) | [Package (npm)](https://www.npmjs.com/package/@azure/ai-projects) |

## Prerequisites

* An Azure subscription - [Create one for free](https://azure.microsoft.com/free/cognitive-services).
* [Node.js LTS](https://nodejs.org/)
* Make sure you have the **Azure AI Developer** [RBAC role](../../../ai-studio/concepts/rbac-ai-studio.md) assigned at the appropriate level.
* Install [the Azure CLI and the machine learning extension](/azure/machine-learning/how-to-configure-cli). If you have the CLI already installed, make sure it's updated to the latest version.

[!INCLUDE [bicep-setup](bicep-setup.md)]

## Configure and run an agent

| Component | Description                                                                                                                                                                                                                               |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agent     | Custom AI that uses AI models in conjunction with tools.                                                                                                                                                                                  |
| Tool      | Tools help extend an agent’s ability to reliably and accurately respond during conversation. Such as connecting to user-defined knowledge bases to ground the model, or enabling web search to provide current information.               |
| Thread    | A conversation session between an agent and a user. Threads store Messages and automatically handle truncation to fit content into a model’s context.                                                                                     |
| Message   | A message created by an agent or a user. Messages can include text, images, and other files. Messages are stored as a list on the Thread.                                                                                                 |
| Run       | Activation of an agent to begin running based on the contents of Thread. The agent uses its configuration and Thread’s Messages to perform tasks by calling models and tools. As part of a Run, the agent appends Messages to the Thread. |
| Run Step  | A detailed list of steps the agent took as part of a Run. An agent can call tools or create Messages during its run. Examining Run Steps allows you to understand how the agent is getting to its results.                                |

First, initialize a new project by running:

```console
npm init -y
```

Run the following commands to install the npm packages required.

```console
npm install @azure/ai-projects
npm install @azure/identity
npm install dotenv
```

Next, to authenticate your API requests and run the program, use the [az login](/cli/azure/authenticate-azure-cli-interactively) command to sign into your Azure subscription.

```azurecli
az login
```

Use the following code to create and run an agent. To run this code, you will need to create a connection string using information from your project. This string is in the format:

`<HostName>;<AzureSubscriptionId>;<ResourceGroup>;<ProjectName>`

[!INCLUDE [connection-string-portal](connection-string-portal.md)]

`HostName` can be found by navigating to your `discovery_url` and removing the leading `https://` and trailing `/discovery`. To find your `discovery_url`, run this CLI command:

```azurecli
az ml workspace show -n {project_name} --resource-group {resource_group_name} --query discovery_url
```

For example, your connection string may look something like:

`eastus.api.azureml.ms;12345678-abcd-1234-9fc6-62780b3d3e05;my-resource-group;my-project-name`

Set this connection string as an environment variable named `PROJECT_CONNECTION_STRING` in a `.env` file.

> [!IMPORTANT] 
> * This quickstart code uses environment variables for sensitive configuration. Never commit your `.env` file to version control by making sure `.env` is listed in your `.gitignore` file.
> * _Remember: If you accidentally commit sensitive information, consider those credentials compromised and rotate them immediately._


Next, create an `index.js` file and paste in the code below:

```javascript
// index.js

import {
  AIProjectsClient,
  DoneEvent,
  ErrorEvent,
  isOutputOfType,
  MessageStreamEvent,
  RunStreamEvent,
  ToolUtility,
} from "@azure/ai-projects";
import { DefaultAzureCredential } from "@azure/identity";
import dotenv from 'dotenv';

dotenv.config();

// Set the connection string from the environment variable
const connectionString = process.env.PROJECT_CONNECTION_STRING;

// Throw an error if the connection string is not set
if (!connectionString) {
  throw new Error("Please set the PROJECT_CONNECTION_STRING environment variable.");
}

export async function main() {
  const client = AIProjectsClient.fromConnectionString(
    connectionString || "",
    new DefaultAzureCredential(),
  );

  // Step 1 code interpreter tool
  const codeInterpreterTool = ToolUtility.createCodeInterpreterTool();

  // Step 2 an agent
  const agent = await client.agents.createAgent("gpt-4o-mini", {
    name: "my-agent",
    instructions: "You are a helpful agent",
    tools: [codeInterpreterTool.definition],
    toolResources: codeInterpreterTool.resources,
  });

  // Step 3 a thread
  const thread = await client.agents.createThread();

  // Step 4 a message to thread
  await client.agents.createMessage(
    thread.id, {
    role: "user",
    content: "I need to solve the equation `3x + 11 = 14`. Can you help me?",
  });

  // Intermission is now correlated with thread
  // Intermission messages will retrieve the message just added

  // Step 5 the agent
  const streamEventMessages = await client.agents.createRun(thread.id, agent.id).stream();

  for await (const eventMessage of streamEventMessages) {
    switch (eventMessage.event) {
      case RunStreamEvent.ThreadRunCreated:
        break;
      case MessageStreamEvent.ThreadMessageDelta:
        {
          const messageDelta = eventMessage.data;
          messageDelta.delta.content.forEach((contentPart) => {
            if (contentPart.type === "text") {
              const textContent = contentPart;
              const textValue = textContent.text?.value || "No text";
            }
          });
        }
        break;

      case RunStreamEvent.ThreadRunCompleted:
        break;
      case ErrorEvent.Error:
        console.log(`An error occurred. Data ${eventMessage.data}`);
        break;
      case DoneEvent.Done:
        break;
    }
  }

  // 6. Print the messages from the agent
  const messages = await client.agents.listMessages(thread.id);

  // Messages iterate from oldest to newest
  // messages[0] is the most recent
  for (let i = messages.data.length - 1; i >= 0; i--) {
    const m = messages.data[i];
    if (m.content && m.content.length > 0 && isOutputOfType(m.content[0], "text")) {
      const textContent = m.content[0];
      console.log(`${textContent.text.value}`);
      console.log(`---------------------------------`);
    }
  }

  // 7. Delete the agent once done
  await client.agents.deleteAgent(agent.id);
}

main().catch((err) => {
  console.error("The sample encountered an error:", err);
});
```

Run the code using `node index.js` and observe.