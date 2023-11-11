# Building a custom client using the API

If none of the built-in integrations meet your requirements, you can deploy your assistant by developing a custom client application that interacts with your users and communicates with the IBM Watson Assistant service.

The Watson SDKs help you write code that interacts with Watson Assistant. For more information about the SDKs, see **Watson SDKs**.

## Setting up the assistant

The example application we will create in this section implements several simple functions to illustrate how a client application interacts with Watson Assistant. The application code will collect input and send it to an assistant, which sends responses the application will display to the user.

If you want to try this example yourself, first you need to set up the simple example assistant that the client will connect to:

1. Download the actions <a target="_blank" href="https://watson-developer-cloud.github.io/doc-tutorial-downloads/assistant/assistant-simple-example-actions.json" download="assistant-simple-example-actions.json">JSON file</a>.
1. **Create an assistant**.
1. In the new assistant, **open the global action settings**. Go to the **Upload/Download** tab and import the actions from the file you downloaded.

The example actions include a *Greet customer* action that asks the customer's name, and simple actions for making and canceling appointments.

## Getting service information

To access the Watson Assistant REST APIs, your application needs to be able to authenticate with IBM Cloud and connect to the assistant in the environment where it is deployed. You'll need to copy the service credentials and environment ID and paste them into your application code. You'll also need the URL for the location of your service instance (for example, `https://api.us-south.assistant.watson.cloud.ibm.com`).

To find this information, follow these steps:

1. Go to the **Environments** page and click the tab for the environment you want to connect to.

1. Click the ![Environment settings icon](images/gear-icon-black.png) icon to open the environment settings.

1. Select **API details** to see details for the environment, including the service instance URL and environment ID. To find the API key, follow the link in the **Service credentials** section.

## Communicating with the Watson Assistant service

Interacting with the Watson Assistant service from your client application is simple. We'll start with an example that connects to the service, sends a single empty message, and prints the output to the console:

```javascript
// Example 1: Creates service object, sends initial message, and
// receives response.

const AssistantV2 = require('ibm-watson/assistant/v2');
const { IamAuthenticator } = require('ibm-watson/auth');

// Create Assistant service object.
const assistant = new AssistantV2({
  version: '2021-11-27',
  authenticator: new IamAuthenticator({
    apikey: '{apikey}', // replace with API key
  }),
  url: '{url}', // replace with URL
});

const assistantId = '{environment_id}'; // replace with environment ID

// Start conversation with empty message
messageInput = {
  messageType: 'text',
  text: '',
};
sendMessage(messageInput);

// Send message to assistant.
function sendMessage(messageInput) {
  assistant
    .messageStateless({
      assistantId,
      input: messageInput,
    })
    .then(res => {
      processResult(res.result);
    })
    .catch(err => {
      console.log(err); // something went wrong
    });
}

// Process the result.
function processResult(result) {
  // Print responses from actions, if any. Supports only text responses.
  if (result.output.generic) {
    if (result.output.generic.length > 0) {
      result.output.generic.forEach( response => {
        if (response.response_type == 'text') {
          console.log(response.text);
        }
      });
    }
  }
}
```

The first step is to create a the service object, a sort of wrapper for the Watson Assistant service.

You use the service object for sending input to, and receiving output from, the service. When you create the service object, you specify the API key for authentication, as well as the version of the Watson Assistant API you are using.

In this Node.js example, the service object is an instance of `AssistantV2`, stored in the variable `assistant`. The Watson SDKs for other languages provide equivalent mechanisms for instantiating a service object.

After creating the service object, we use it to send a message to the assistant, using the stateless `message` method. In this example, the message is empty; we just want to trigger the *Greet customer* action to start the conversation, so we don't need any input text. We then print any text responses returned in the `generic` array in the returned output.

Use the `node <filename.js>` command to run the example application.

**Note:** Make sure you have installed the Watson SDK for Node.js using `npm install ibm-watson`.

Assuming everything works as expected, the assistant returns the output from the assistant, which the app then prints to the console:

```text
Welcome to the Watson Assistant example. What's your name?
```

This output tells us that we have successfully communicated with the assistant and received the greeting message specified by the *Greet customer* action. But we don't yet have a way of responding to the assistant's question.

## Processing user input

To be able to process user input, we need to add a user interface to our client application. For this example, we'll keep things simple and use standard input and output. We can use the Node.js prompt-sync module to do this. (You can install prompt-sync using `npm install prompt-sync`.)

```javascript
// Example 2: Adds user input.

const prompt = require('prompt-sync')();
const AssistantV2 = require('ibm-watson/assistant/v2');
const { IamAuthenticator } = require('ibm-watson/auth');

// Create Assistant service object.
const assistant = new AssistantV2({
  version: '2021-11-27',
  authenticator: new IamAuthenticator({
    apikey: '{apikey}', // replace with API key
  }),
  url: '{url}', // replace with URL
});
  
const assistantId = '{environment_id}'; // replace with environment ID

// Start conversation with empty message
messageInput = {
  messageType: 'text',
  text: '',
};
sendMessage(messageInput);

// Send message to assistant.
function sendMessage(messageInput) {
  assistant
    .messageStateless({
      assistantId,
      input: messageInput,
    })
    .then(res => {
      processResult(res.result);
    })
    .catch(err => {
      console.log(err); // something went wrong
    });
}

// Process the result.
function processResult(result) {

  // Print responses from actions, if any. Supports only text responses.
  if (result.output.generic) {
    if (result.output.generic.length > 0) {
      result.output.generic.forEach( response => {
        if (response.response_type === 'text') {
          console.log(response.text);
        }  
      });
    }
  }

  // Prompt for the next round of input unless skip_user_input is true.
  let newMessageFromUser = '';
  if (result.context.global.system.skip_user_input !== true) {
    newMessageFromUser = prompt('>> ');
  }
  
  if (newMessageFromUser !== 'quit') {
    newMessageInput = {
      messageType: 'text',
      text: newMessageFromUser,
    }
    sendMessage(newMessageInput);
  }
}
```

This version of the application begins the same way as before: sending an empty message to the assistant to start the conversation.

The `processResult()` function displays the text of any responses received from the assistant. It then prompts for the next round of user input.

**Tip:** The example checks for the global context variable `skip_user_input` and prompts for user input only if this variable is not set to `true`. The `skip_user_input` variable is set by the assistant in some situations where no user input is needed (for example, if the assistant has called an external service but is still waiting for the result). It's good practice always to make this check before prompting for user input.

Because we need a way to end the conversation, the client app is also watching for the literal command `quit` to indicate that the program should exit.

But something still isn't right:

```
Welcome to the Watson Assistant example. What's your name?
>> Robert
I'm afraid I don't understand. Please rephrase your question.
>> I want to make an appointment.
What day would you like to come in?
>> Thursday
I'm afraid I don't understand. Please rephrase your question.
>>
```

The assistant is starting out with the correct greeting, but it doesn't understand when you tell it your name. And if you tell it you want to make an appointment, the correct action is triggered; but once again, it doesn't understand when you answer the follow-up question.

This is happening because we are using the stateless `message` method, which means that it is the responsibility of our client application to maintain state information for the conversation. Because we are not yet doing anything to maintain state, the assistant sees every round of user input as the first turn of a new conversation. Because it has no memory of asking a question, it tries to interpret your answer as a new question or request.

## Maintaining state

State information for your conversation is maintained using the *context*. The context is an object that is passed back and forth between your application and the assistant, storing information that can be preserved and updated as the conversation goes on. Because we are using the stateless `message` method, the assistant does not store the context, so it is the responsibility of our client application to maintain it from one turn of the conversation to the next.

The context includes a session ID for each conversation, as well as a counter that is incremented with each turn of the conversation. The assistant updates the context and returns it with each response. But our previous version of the example did not preserve the context, so these updates were lost, and each round of input appeared to be the start of a new conversation. We can fix that by saving the context and sending it back to the assistant each time.

In addition to maintaining our place in the conversation, the context can also contain action variables that store any other data you want to pass back and forth between your application and the assistant. This can include persistent data you want to maintain throughout the conversation (such as a customer's name or account number), or any other data you want to track (such as the contents of a shopping cart or user preferences).

```javascript
// Example 3: Preserves context to maintain state.

const prompt = require('prompt-sync')();
const AssistantV2 = require('ibm-watson/assistant/v2');
const { IamAuthenticator } = require('ibm-watson/auth');

// Create Assistant service object.
const assistant = new AssistantV2({
  version: '2021-11-27',
  authenticator: new IamAuthenticator({
    apikey: '{apikey}', // replace with API key
  }),
  url: '{url}', // replace with URL
});

const assistantId = '{environment_id}'; // replace with environment ID

// Start conversation with empty message
messageInput = {
  messageType: 'text',
  text: '',
};
context = {};
sendMessage(messageInput);

// Send message to assistant.
function sendMessage(messageInput, context) {
  assistant
    .messageStateless({
      assistantId,
      input: messageInput,
      context: context,
    })
    .then(res => {
      processResult(res.result);
    })
    .catch(err => {
      console.log(err); // something went wrong
    });
}

// Process the result.
function processResult(result) {

  let context = result.context;

  // Print responses from actions, if any. Supports only text responses.
  if (result.output.generic) {
    if (result.output.generic.length > 0) {
      result.output.generic.forEach( response => {
        if (response.response_type === 'text') {
          console.log(response.text);
        }  
      });
    }
  }

  // Prompt for the next round of input unless skip_user_input is true.
  let newMessageFromUser = '';
  if (result.context.global.system.skip_user_input !== true) {
    newMessageFromUser = prompt('>> ');
  }

  if (newMessageFromUser !== 'quit') {
    newMessageInput = {
      messageType: 'text',
      text: newMessageFromUser,
    }
    sendMessage(newMessageInput, context);
  }
}
```

The only change from the previous example is that we are now storing the context received from the assistant in a variable called `context`, and we're sending it back with the next round of user input:

```javascript
  assistant
    .messageStateless({
      assistantId,
      input: messageInput,
      context: context,
    })
```

This ensures that the context is maintained from one turn to the next, so the Watson Assistant service no longer thinks every turn is the first:

```
Welcome to the Watson Assistant example. What's your name?
>> Robert
Hi, Robert! How can I help you?
>> I want to make an appointment.
What day would you like to come in?
>> Next Monday
What time works for you?
>> 10 AM
OK, Robert. You have an appointment for 10:00 AM on Sep 12. See you then!
```

Success! The application now uses the Watson Assistant service to understand natural-language input, and it displays the appropriate responses.

This simple example illustrates how you can build a custom client app to communicate with the assistant. Of course, a real-world application would use a more sophisticated user interface, and it might integrate with other applications such as a customer database or other business systems. It would also need to send additional data to the assistant, such as a user ID to identify each unique user. But the basic principles of how the application interacts with the Watson Assistant service would remain the same.

## Using the v1 runtime API

Using the v2 API is the recommended way to build a runtime client application that communicates with the Watson Assistant service. However, some older applications might still be using the v1 runtime API, which includes a similar method for sending messages to the workspace within a dialog skill. Note that if your app uses the v1 runtime API, it communicates directly with the workspace, bypassing the skill orchestration and state-management capabilities of the assistant.

For more information about the v1 `/message` method and context, see the [v1 API Reference](https://cloud.ibm.com/apidocs/assistant/assistant-v1#message).

*&copy; 2019, 2022 IBM Corporation*
