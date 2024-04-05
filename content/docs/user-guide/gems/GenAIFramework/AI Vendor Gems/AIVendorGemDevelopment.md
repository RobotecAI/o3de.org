---
linkTitle: AI Vendor Gem Development Guide
title: AI Vendor Gem Development Guide
description: Learn how to develop your own AI Vendor Gem for the Generative AI Framework.
weight: 400
---

Each Vendor Gem is designed as a typical [O3DE Code Gem](/docs/user-guide/programming/gems/code-gems/) and should consist of two parts:
- The communication component that interacts with the AI service.
- The model configuration component that allows the user to configure the specific parameters of the AI service.
Each component is meant to be communicated using [EBusses](/docs/user-guide/programming/components/ebus-integration/).

## Examples
Available examples of AI Vendor Gems can be found in this [AI Vendor Gems repository](https://github.com/RobotecAI/ai-vendors-gems/).

## Provided interfaces
The GenAIFramework Gem provides two interfaces that the AI Vendor Gem should implement:
- `AIServiceRequesterBus` - This interface is used to communicate with the AI service.
- `AIModelRequestBus` - This interface is used to create the requests sent to the AI service with the specified user configuration.  

In addition for these components to be recognized by the GenAIFramework, they should be registered using the `GenAIFramework::SystemRegistrationContext`.
This registration context provides `RegisterModelConfiguration` for registering the components implementing the `AIModelRequestBus` and `RegisterAIServiceRequester` for registering the components implementing the `AIServiceRequesterBus`.  

To learn more about the reflection system, see the [Reflection Contexts documentation](/docs/user-guide/programming/components/reflection/).

### AIServiceRequesterBus
This interface requires the implementation of the `SendRequest` function. This function should send the request to the AI service. A callback is used when the response is received.

### AIModelRequestBus
This interface requires the implementation of the `PrepareRequest`, `ExtractResult` and `SetModelParameter` functions:
- `PrepareRequest` - This function should prepare the request to be sent to the AI service. As a parameter it takes a prompt to be sent to the AI service. The result should be a JSON formatted string containing all of the required parameters for the AI service.
- `ExtractResult` - This function should extract the result from the response received from the AI service. As a parameter it takes the JSON formatted response from the AI service. This functions returns the extracted result.
- `SetModelParameter` - This function should set the parameters of the AI service. As a parameter it takes the name of the parameter and the value to be set.

## Development steps
### Creating a new Gem
To get started with developing your own AI Vendor Gem, create a [O3DE Code Gem](/docs/user-guide/programming/gems/code-gems/).

### Implementing the interfaces
Create two components that implement the `AIServiceRequesterBus` and `AIModelRequestBus` interfaces. See the [Provided interfaces](#provided-interfaces) section for more information on the implementation details.  
Each component should be reflected with appropriate methods in `GenAIFramework::SystemRegistrationContext`. The example below shows how to register `AIModelRequest` component.
```c++
if (auto registrationContext = azrtti_cast<GenAIFramework::SystemRegistrationContext*>(context))
{
    registrationContext->RegisterModelConfiguration<YourComponentName>();
}
```

> 💡 If a component is not reflected, it will not be recognized by the GenAIFramework and subsequently not available for creation in the GenAIFramework widget.

After building the Gem, these components should be available in the GenAIFramework widget.  
TODO insert example image
