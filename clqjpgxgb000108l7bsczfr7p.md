---
title: "Serverless social login with AWS Cognito"
seoTitle: "Serverless OAuth with Amazon Cognito"
seoDescription: "Simple serverless solution to add Social Login to your app with Amazon Cognito"
datePublished: Sun Dec 24 2023 16:30:24 GMT+0000 (Coordinated Universal Time)
cuid: clqjpgxgb000108l7bsczfr7p
slug: serverless-social-login-with-aws-cognito
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/ZYLmudR28SA/upload/b8e67f93d0f65e547d047b89d9e46698.jpeg
tags: aws, oauth, serverless, social-login, cognito

---

*Disclaimer: This is not a step-by-step guide, just my trade-off analysis on using Amazon Cognito to provide social login for your app and some pitfalls I found in my experience.*

In this article, I'll show you my serverless solution to add social identity providers as a login option for web and mobile applications, based on managed services and native integrations, and how I mitigated some issues I encountered.

## Context

Let's assume you have an application for which your users do not have to register, but can log in with their social identity.

If you 're wondering why, think about it:

* registration could be a barrier to entry for users as it requires more steps and sharing of data
    
* most internet users have at least one social identity. All mobile users have at least one (Google identity for Android users, Apple identity for Apple users)
    
* it is very easy for users to access your app if most of the login is done without a password
    
* you can receive users data from social providers, if users allow your app to give their data to your app.
    

The most popular social IdPs are Facebook, Google, Apple, Amazon, LinkedIn, Github and many others.

Considering that every IdP should implement the OpenID Connect standard (we'll come back to this later...), which is a layer above the OAuth2 standard, and that every IdP requires some configuration, let's explore some options.

## Option 1: Native integration

Every one has its own SDK and apis to integrate natively, so you can code in your app the integration for IdPs you want to use.

![direct integration with IdP's SDK](https://cdn.hashnode.com/res/hashnode/image/upload/v1702570245136/d462c300-30e2-4a59-bcc1-8773f7710c1e.png align="center")

### Pros

* fine-grained control over each individual IdP integration. Since each IdP is natively integrated, you can customise the specific UX via configuration and handle IdP requests that are not included in the OAuth standard (we'll get to that later...)
    
* direct integration, no intermediary, straightforward architecture. You can rely on robust implementations (Google / Facebook / Amazon provides good code in their SDK) and IdP resilience and H-A.
    
* Cost-effective: usually IdPs provide a free-tier for their api, so there aren't any costs from that side.
    

### Cons

* Difficult to scale: Each IdP has its own SDK and its own customs (someone said "standard?"), and it takes a lot of code is required to handle them. Even if you put your authentication logic into a library, you have to distribute it to all clients to get a change.
    
* Hard to test / troubleshoot: more code, more tests. Moreover, different integrations require you to know IdP customs.
    

## Option 2: use an OAuth Provider

Since Social IdP adhere to a standard, it's easy to abstract the specific implementation (SDK) and work with interfaces, integrating with an OAuth 2 service provider.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702912629290/ecb258f3-eb97-4010-94c3-4b4ba91d3363.png align="center")

### Pros

* Just one integration, between your client and di OAuth identity platform. Less code, less test, less releases, more speed.
    
* Easy to scale: you can add/remove IdP without impacting clients (see previous bullet)
    
* Authentication flow configuration and governance is now centralised. You can create consistent auth flow regardless the specific IdP you support, and you can monitor it and gather metrics and statistics in one place.
    
* You build your auth flow on standards.
    
* There are Identity Platform as a service out there (AWS Cognito, Auth0, Google Firebase and many others)
    

### Cons

* Your integration choices are limited to IdPs supported by your OAuth provider.
    
* Your system complexity is higher, since you add components to it.
    
* The OAuth provider could be a single point of failure. If it is not available, you cannot offer authentication to your customers. Therefore, you need to think carefully about the reliability and scaling of your OAuth provider.
    

## My choice: Option 2 with AWS Cognito

I'm aware that you may have many constraints and to be brief, I cannot list them all: given my context, I went with option 2 and used AWS Cognito as OAuth provider. I did a spike on Auth0 and some other services.

> I decided to accept the constraints and costs of Cognito in exchange for a low-code implementation and easy setup, in other words for faster delivery, because I wasn't sure if it would be worth it.

Here my actual implementation

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702913119260/edd0ce2e-c73c-40fa-ba08-bc40003cb45d.png align="center")

All you need is to:

* configure your integration on Social Provider side. Here a reference for each of provider i integrated with
    
    * [Google](https://ryandam9.medium.com/using-google-as-an-identity-provider-in-aws-cognito-acddfb58fad)
        
    * [Facebook](https://victorhzhao.medium.com/add-social-login-to-aws-cognito-user-pool-facebook-94a2cee5136e)
        
    * [Apple](https://jainsameer.medium.com/react-native-social-sign-in-with-apple-and-amplify-6c803b2971d6)
        
* configure Cognito integration. [Here AWS Doc for each supported providers](https://docs.aws.amazon.com/cognito/latest/developerguide/external-identity-providers.html)
    
* [Integrate your application with Amazon Cognito](https://aws.amazon.com/cognito/dev-resources/?nc1=h_ls). Cognito provides an [hosted ui](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-integration.html) for the login page, but you can create your own.
    

## Pitfalls: things to be careful about

Here I list some of the pifalls I have encountered in this integration. This is not an exhaustive list of what goes wrong with Amazon Cognito and the social login flow, but again my personal experience or in other words things I found during my working days.

### Watch out for Cognito limits

Serverless does not mean infinite, and Cognito is one of the services that best demonstrates this.

In one sentence: Cognito's scaling policy is not designed for spiky patterns.

The scaling pattern is (reasonably) tied to the size of your user pool: the more users, the more TPS provided.

But, and here comes the first pitfall, the first threshold is up to 1 million users. From 1 to 999999 users, you have the same TPS.

This means that if your login pattern is fairly consistent, you probably won't have any problems. However, if your login pattern is spiky, perhaps because your app is tied to certain time periods in some way, your app will struggle with a lot of throttling errors from Cognito.

These diagram show successful federation logins and throttling errors:

![Cognito success login](https://cdn.hashnode.com/res/hashnode/image/upload/v1703411639152/b6027e66-8b18-402b-981c-ecce4697d4de.png align="center")

![Cognito Throttling errors](https://cdn.hashnode.com/res/hashnode/image/upload/v1703411372971/361a812c-244f-4402-b2bb-be9822d2c3d2.png align="center")

i split into two distinct diagrams for better visualisation, but i want to point out that

* around 20:50 i had ~7K throttling errors and ~1.5K of success (total requests: ~8.5K)
    
* around 21:20 i had ~6K throttling errors and ~1.4K success (total requests: ~7.5K)
    
* around 22:30 i had ~1.3K success with ZERO throttling errors
    

Cognito TPS calculation rules can be found [at this specific section of Cognito docs](https://docs.aws.amazon.com/cognito/latest/developerguide/limits.html), and you have to carefully consider them.

As you can see from the successfully login metric diagram, handling the throttling exception in your app can mitigate the user impact: they would be able to successfully login anyway, but waiting a little bit more.

> ***I decided that it could be acceptable, and i traded it for easy setup and integration with Social Providers.***

Since this decision would impact our customer experience i tried to mitigate it as much as possible, for instance sending push notifications before traffic spikes to encourage users to log in and spread log in requests.

### Standards are not prescriptive

I love standards, everybody should love them in engineering.

Unfortunately, sometimes for good reasons and sometimes not, giants have bias to force standards a little bit.

Apple, i'm pointing my finger at you!

First, Apple's guidelines require you to log in to Apple if you want to distribute your app in the Apple Store and your app has a social login feature. That may be a bit rude, but it's fair.

Again,Apple prescribe you that the "User cancellation" function must be accessible and clear. That is also fair.

And here Apple does not adhere to the OAuth standard: if an Apple user allows Apple to share their data with your app, some kind of association between your app and the user also takes place in the Apple system, and if a user wants to cancel from your app (also known as your user pool), this association should also be removed.

To do that, you have to invoke Apple apis to:

* generate a valid access or refresh token.
    
* invalidate the freshly generated token.
    

[Sounds weird, but this is exactly what this doc page prescribes.](https://developer.apple.com/documentation/sign_in_with_apple/revoke_tokens)

And, guess what? Cognito doesn't handle it.

Even if Cognito could handle it because it has all the information it needs, especially the private key you created on the Apple side and provided to Cognito to request the tokens, that's reasonable from a product perspective: Cognito adheres to standards and can't track every specific implementation.

But it does mean that Apple won't include your app in the store if you don't take care of it.

So let's take a look at how to implement it.

You can't implement it in the app: i used Cognito to decouple the app from auth providers, and i don't want to violate that requirement. Besides, you don't want to store your private key on the device, do you?

So you need to implement it on the backend side. My first idea was to use events to respond to the Cognito event to delete a user and trigger a lambda that calls the Apple api to delete the user on the Apple side.

As far as I know, Cognito today has

* [Lambda triggers](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools-working-with-aws-lambda-triggers.html#cognito-user-pools-lambda-trigger-event-parameter-shared): user deletion not supported
    
* [Cloudtrail tracks all management api calls](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-info-in-cloudtrail.html), and user cancellation is a management api. But Cloudtrail event doesn't have any reference to actual user (and it saved my day in an audit session, but this is another story)
    
* [Cognito Sync](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-events.html): it seems to handle user deletion. Quoting:
    
    > To remove a record, either set the `op` to `remove`, or set the value to null.
    

This is how it looks like:

![Apple user cancellation w/ Cognito Sync](https://cdn.hashnode.com/res/hashnode/image/upload/v1703423682433/7a7f0717-55d9-43b9-966a-0c60e6d9c39c.png align="center")

I see two problems here:

* first, you have to put your Apple's private key in Cognito and in Secret Manager. Cognito can't retrieve it from Secret Manager. I raised this issue to Cognito team, keep you posted on this.
    
* second, Cognito user cancellation and Apple user cancellation are asynchronous: what if it success on Cognito side and than fails on Apple side? User wont be in our Cognito user pool anymore, so we can't rollback the operation. So you need to handle failures, and to handle it you need to store it. Let's add a DLQ for our deletion lambda
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1703424275924/79d20cd1-a18d-4ff3-829a-16905768712f.png align="center")

After saving, you must analyse why the deletion failed and try again. How long can this take? It depends on the cause and your process, but until you've done that, users will still see their user associated with your device, and I'm not sure Apple would like it and approve your app submission.

You need to reverse the order of deletion, first on the Apple side and then on the Cognito side. If the Apple deletion fails, you can send an error message to the user and inform he/she that the deletion cannot be performed and they should try again later.

In the case of a Cognito error, you will have to do this later, but at least the user will not see that their user is linked to your app and Apple should be satisfied and approve your request.

Let's see how it looks like

![User deletion with custom api](https://cdn.hashnode.com/res/hashnode/image/upload/v1703425150169/dad05420-73e0-42b8-9875-9356799194bc.png align="center")

I still see two problems here:

* Again , you have to put your Apple's private key in Cognito and in Secret Manager.
    
* Your app now is integrated with two systems: Cognito for Sign-in operation and your custom api for user deletion
    

Both solutions somehow solve the problem and both raise new concerns, so I had to opt for the less bad one.

> I decided to implement a custom api for Apple user deletion because it can be implemented just in half our code base (not for Android version of the app), the integration is quite simple and Apple would be happy with this solution, but probably not with the alternative solution. Still an error handling mechanism still need to be implemented to catch Cognito deletion errors and to recover them.

## Wrap up

I have shown you my solution to real-world problems and how you can make informed decisions by carefully weighing trade-offs between different solutions that best fit your context and constraints.

In other words, the daily work of an architect, simplified.

Architectures need to evolve as the context and constraints change over time. So always design your solutions so that they can easily evolve with you.

I hope it was useful for you!

Bye 👋!