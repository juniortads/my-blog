+++
date = "2018-03-18"
title = "Serverless (Azure Functions) + Chatbots [en-US]"
+++

`Have you already talked with a chatbot to solve anything?`

![chatbot](/images/chatbot-1.gif)

Definitely, since 2016 they have invaded our daily lives and that is amazing, because some tasks that you needed to talk with a human, today you can talk with a chatbot. Airline tickets, ordering from restaurants and so on.

However, if you are a software developer like me you have many framework options to help build a chatbot.

#### Check out this comparative table

![chatbot](/images/chatbot-2.png)

Another subject that has come out over the last years and I believe you have already heard, it’s “serverless”.
`FaaS` is a form of event based computing. Basically, you configure functions to be triggered by specific events. You have many benefits: cost, time to market, scalability etc. So you want to implement your code using serverless I would suggest:

> There are other serverless providers. The implementation details are going to be different depending on which one you use.

[Introduction to Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-overview)

[AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/index.html)

Okay, but have you thought about building a chatbot using serverless?

![chatbot](/images/chatbot-3.gif)

Recently, Microsoft made BOT Framework avaiable.

BOT Framework is a service based on SDK and your connector. When you build a BOT you can use many services like language Understanding Intelligent Service (LUIS), Speech API and most cognitive services. Even more interesting, it can be integrated across multiple channels. Then, let’s get started little bit code.

#### Set up and Create a New Project

In Visual Studio 2017 you can look for: new project -> cloud -> Azure functions. Don’t forget you need Azure SDK when you install VS.

![chatbot](/images/chatbot-4.png)

Then, after you create new function you can install new packages.

![chatbot](/images/chatbot-5.png)

![chatbot](/images/chatbot-6.png)

This is very important, because that packages enables authentication from BOT to channels and LUIS.ai.

```csharp
public static class HelloBot
{
        [FunctionName("HelloBot")]
        public static async Task<HttpResponseMessage> Run([HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)]HttpRequestMessage req, TraceWriter log)
        {
            log.Info("C# HTTP trigger function processed a request...");

            using (BotService.Initialize())
            {
                string jsonContent = await req.Content.ReadAsStringAsync();
                var activity = JsonConvert.DeserializeObject<Activity>(jsonContent);

                if (!await BotService.Authenticator.TryAuthenticateAsync(req, new[] { activity }, CancellationToken.None))
                {
                    return BotAuthenticator.GenerateUnauthorizedResponse(req);
                }

                if (activity != null)
                {
                    switch (activity.GetActivityType())
                    {
                        case ActivityTypes.Message:
                            await Conversation.SendAsync(activity, () => new GreetingDialog());
                            break;
                        case ActivityTypes.ConversationUpdate:
                            var client = new ConnectorClient(new Uri(activity.ServiceUrl));
                            IConversationUpdateActivity update = activity;
                            if (update.MembersAdded.Any())
                            {
                                var reply = activity.CreateReply();

                                var newMembers = update.MembersAdded?.Where(t => t.Id != activity.Recipient.Id);
                                foreach (var newMember in newMembers)
                                {
                                    reply.Text = "Welcome";
                                    if (!string.IsNullOrEmpty(newMember.Name))
                                    {
                                        reply.Text += $" {newMember.Name}";
                                    }
                                    reply.Text += "!";
                                    await client.Conversations.ReplyToActivityAsync(reply);
                                }
                            }
                            break;
                        case ActivityTypes.ContactRelationUpdate:
                        case ActivityTypes.Typing:
                        case ActivityTypes.DeleteUserData:
                        case ActivityTypes.Ping:
                        default:
                            log.Error($"Unknown activity type ignored: {activity.GetActivityType()}");
                            break;
                    }
                }
            }
            return req.CreateResponse(HttpStatusCode.Accepted);
        }
    }
```

---

```csharp
public class GreetingDialog : IDialog<object>
{
      public async Task StartAsync(IDialogContext context)
      {
          await context.PostAsync("Hello!!");
          context.Wait(MessageReceivedAsync);
      }

      public virtual async Task MessageReceivedAsync(IDialogContext context, IAwaitable<IMessageActivity> argument)
      {
          var message = await argument;
          await context.PostAsync($"You said {message.Text}");
          context.Wait(MessageReceivedAsync);
      }
}
```
#### Deploy and Send a Test Request

Basically, you have some options to deploy your function. For this article, I’ve used “Deployment Options” on configured features in Azure Portal.
It is very easy and you need to configure deployment option: choose source you prefer then project and finally branch.

![chatbot](/images/chatbot-7.png)

![chatbot](/images/chatbot-8.png)

You can get url your function on azure portal.

![chatbot](/images/chatbot-9.png)

So, completing these steps, you can a test your bot.

![chatbot](/images/chatbot-10.png)

[Debug with the Emulator](https://docs.microsoft.com/en-us/azure/bot-service/bot-service-debug-emulator?view=azure-bot-service-4.0&tabs=csharp)

#### Conclusion

I believe Serverless and Chatbots are the next big thing in software development, and little by little the way we interact is changing, so why not combine that with benefits of cost and scalability.

`What do you think about that?`

![chatbot](/images/chatbot-done.gif)