## I Can Alexa and So Can You
Have you ever spent an afternoon shouting at a computer and wishing it would just do the thing you said? Have you giggled to yourself after making a voice synthisizer repeat "doo doo" for minutes straight? Have you wished that there was an intersection of the two preceding things packaged in a handy JavaScript SDK?

Well, dear reader, I have good news about your increasingly specific wishes. Amazon has a Node SDK for their Alexa Skills Kit, and it significantly reduces the learning curve for getting your hands dirty with Alexa Skills. To go over the basics, we're going to put together a simple dice-rolling skill.

### Getting Started
Clone this repo and run 
```npm install```
from within the src folder. You'l also want to install a quick and dirty Alexa skill tester globally, so run
```npm install -g alexa-skill-test```
after the first part's done. 
Next, you'll need to have a couple accounts for publishing purposes: an [Amazon developer account](https://developer.amazon.com) and an [AWS account](https://aws.amazon.com/).

### Alexa Basics

#### Words Words Words
There are three pieces of vocabulary that are pretty essential to understanding Alexa: utterances, intents, and slots.

##### Utterances
These are the words a user says to Alexa to ask her to do something or provide a response to a question she asked. If a user wanted to use the Domino's skill to order a pizza, they might say:

> Alexa, ask Domino's to order me a pizza

But similar utterances could have the same result:

>ask Domino's to get me a pizza

>open Domino's and order a pizza

>tell Domino's that unless I have a pizza on my doorstep in the next 15 minutes I swear to God

etc.

When creating a skill, you'll give Alexa a list of sample utterances. They come attached to a particular intent (which we'll get to momentarily) that are used by Alexa to generate a larger list of possible utterances. It should look something like this:

```
OrderPizzaIntent i want a pizza
OrderPizzaIntent order a pizza
OrderPizzaIntent get me a pizza
```

These utterances should be a varied cross-section of possble ways for a user to make a request. You're not looking to account for every possible change in wording - that's Alexa's job.

##### Intents
Intents are the ultimate request that a user's utterence maps to. That is, ordering a pizza, changing an address, or opening the pod bay doors. As mentioned above, a single intent can be triggered by a variety of similar utterences - both by the sample utterences and their derivations.

Along with said sample utterences, you'll create an intent schema that tells Alexa which intents you're using. Alexa has a number of built-in intents that don't require sample utterences (things like an intent to ask for help or an intent to cancel) that you can take advantage of, but you're going to need some number of custom intents to make your skill go. The good news is that this is as easy as adding a new intent name to your schema and listing which slots you're using. We'll get to what that looks like once we've covered the final piece, which brings us to...

##### Slots
Slots are the user-supplied arguments for an intent. In other words, they're the extra details a user says with the utterance to give Alexa more information about what the user's request. For example,
> order me a {*pepperoni*} pizza

> which movies are playing at the {*majestic bay*}

> set phasers to {*kill*}

Slots aren't entirely open-ended; you provide a list of values that the Alexa's speech recognition is weighted towards, but it's not entirely constraied to said list (so error-handling becomes important for when a user tries to set phasers to pepperoni). Amazon provides a number of default slots, covering such things as numbers, city names, and comic book titles, but if you have needs that aren't covered by the defaults, you'll have to provide your own slot type and values to the skill (which is about as easy as making a custom intent).

So, let's take a look at that schema:
```javascript
{"intents": [
    {"intent": "VolumeIntent", "slots": [{"name": "Volume", "type": "AMAZON.NUMBER"}]},
    {"intent": "AMAZON.YesIntent"},
    {"intent": "AMAZON.NoIntent"},
  ]
}
```
Here, we're taking advantage of a couple default intents (which you may recognize from the all-caps AMAZON), and creating one of our own. An intent can have multiple slots - we just add to the array. 

#### Boilerplate
Now that we've got terminology out of the way, let's take a look a what it takes to get started.
```javascript
const Alexa = require('alexa-sdk')

exports.handler = function (event, context, callback) {
  const alexa = Alexa.handler(event, context, callback)

  alexa.registerHandlers(handlers)
  alexa.execute()
}
```
So, what's going on after we require in Alexa? First, we're creating an `alexa` object to work with, then we register our intent handlers using, appropriately enough, `registerHandlers()`, and finally, we tell the `alexa` object to execute the skill with `execute()`. That's it! As you may have surmised, most of the work is going to be done by our handlers. In this example, we're not going to get particularly complex, but it's worth noting that we can register more than one handler at once, like so:
```javascript
alexa.registerHandlers(handler1, handler2, handler3)
```

#### HANDLE IT
Let's look at some of the intents handlers our handlers object might consist of.
```javascript
const handlers = {
  'LaunchRequest': function () {
    const speech = 'Welcome to This Example! You can ask me questions like, "what?" or "just, why?". ' +
    'What would you like to know?'
    const reprompt = 'Sorry, I didn\'t catch that. Ask me the thing again.'

    this.emit(':ask', speech, reprompt)
  }
}
```
Let's focus on the `this.emit()` first, because it's how responses to the user are generated and sent. When we emit using `:ask`, we wait for the user to provide more input; when we emit with `:tell`, we finish the session and don't care about anything else the user might have to say. As you can see here, when we use `:ask`, we provide an initial bit of speech and an extra piece to reprompt the user if they don't say anything that can be mapped to an intent before Alexa gets tired of waiting. With `:tell`, we only need the inital string, because we don't have to prompt the user for any more information.

`:ask` and `:tell` aren't the only arguments `this.emit()` can take - there are things like `:tellWithCard`, which will also create a "card" that appears on the user's Alexa app, or `:confirmIntent`, for times when you want to be extra sure that the user wants to order a pizza, and not, say, a healthy salad. You can even emit other intent handlers - more on that in a moment. 

"LaunchRequest" is a special handler that's used if the user starts using a skill without a specific invocation (by saying something like "open [skill name]", for example). This is required by the certification process, and should say the skill's name and provide a *consise* overview of what the user can do, hopefully with an example prompt or two.

Let's take a look a couple other required required handlers - stop and cancel.
```javascript
'AMAZON.StopIntent': function () {
  this.emit('SessionEndedRequest')
},
'AMAZON.CancelIntent': function () {
  this.emit('SessionEndedRequest')
},
'SessionEndedRequest': function () {
  this.emit(':tell', 'Goodbye!')
}
```
Hey, it's more default intents! `StopIntent` and `CancelIntent` will catch a wide variety of user utterences to terminate the skill and route them to our `SessionEndedRequest` handler, which just uses a `:tell` to offer a cordial farewell and stop listening for input, ending the skill. Remember when I said you could emit other handlers? It's as easy as using the handler name as the first argument. 

Asute observers might ask why we're using `StopIntent` and `CancelIntent` interchangably and what the difference is between them. In short, they correspond to different utterences, but unless you have a good reason for differentiating them, Amazon suggests that you treat them the same. If you really want the words "stop" and cancel" to mean different things to your skill, that's a thing you can do, though it may make getting through certification a little harder.

The final required intent is `AMAZON.HelpIntent`, which is where you can offer some more in-depth instruction to your user and provide a few more example utterences to get them started. It doesn't use any different syntax from what we've looked at already, so without further ado, let's get to the skill building!

### Magic Dice Machine
Alright, take a look at the index.js in the src folder. You'll notice a lot of the intents mentioned above are already there, but our custom intent - RollIntent - is blank. If you poked around the intent schema and sample utterances, you might have noticed that we're expecting the user to give us a number in the `times` slot. So, how do we get that number? This is maybe a little less intutive. Our slot value lives off of the event object, like so:
```javascript
this.event.request.intent.slots.slotName.value
```

So, given that, we should be able to implement a roll intent for 6-sided dice. Get the slot value, do some math, and emit a response! Run `alexa-skill-test` in your console to check that your skill is handling input correctly. You will have to manually type in the intent and slot names, but such is the price of glory. 

.

.

.

.

.

.

.

When you're done, you should have something like this:
```javascript
'RollIntent': function () {
  const times = this.event.request.intent.slots.times.value
  let value = 0

  for (let i = 0; i < times; i++) {
    value += Math.floor(Math.random() * 6) + 1
  }

  this.emit(':tell', `Your roll is... ${value}!`)
}
```

That's all well and good, but the skill might be more useful if we could specify the number of sides on the die, too, just in case we decide to take up D&D. Take a stab at updating the intent schema, utterances, and index.js with a new `sides` slot, and then we'll reconvene.

.

.

.

.

.

.

.

First up, let's take a look at the intent schema. If you copied and pasted here, I forgive you.
```javascript
{"intent": "RollIntent", "slots": [{"name": "times", "type": "AMAZON.NUMBER"}, {"name": "sides", "type": "AMAZON.NUMBER"}]}
```
Straightforward, yeah? The sample utterances leave a bit more to taste, but you should make sure that you're including both of the slots.
```
RollIntent roll a {sides} sided die {times} times
RollIntent roll {times} d {sides}
RollIntent roll a d {sides} {times} times
RollIntent {times} d {sides}
```
And finally, the handler!
```javascript
'RollIntent': function () {
  const times = this.event.request.intent.slots.times.value
  const sides = this.event.request.intent.slots.sides.value
  let value = 0

  for (let i = 0; i < times; i++) {
    value += Math.floor(Math.random() * sides) + 1
  }

  this.emit(':tell', `Your roll is... ${value}!`)
}
```
Now that we've got a simple skill running, let's take a look at getting it up and running online so we can listen to the fruits of our labor. 

### Submission
Let's go login to [the Amazon developer site](https://developer.amazon.com) and click on Alexa on the nav bar. Follow the Alexa Skill Kit link, and then go ahead a click on "add new skill". You should see this:
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/1.png">
Let's set the display name and invocation name (which are the actual word(s) a user will use to trigger your skill) to "High Roller" and move on to the interaction model.
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/2.png">
Here are spaces for your intent schema, custom slots if you need them (which we don't), and sample utterances, which comprise the interaction model. Go ahead and paste in your intent schema and utterances, and we'll move on. 

As an aisde, the beta skill builder at the top is pretty cool alternative way to go about building your interaction model and is well worth playing around with, but for the time being, we're keeping things simple.

Now, we need to head to [AWS](https://aws.amazon.com/) to put our skill into a Lamda function. Once you've signed in, click on Services and type Lambda into the search box. 
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/3.png">

Make sure your region (in the upper right) is set to US East(North Virginia), because it's the only North American region that supports our Alexa-Lambda nonsense. 

<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/4.png">

We're going to follow this up with that big blue "Create a Lambda Function" button, and then click on the blank function template.
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/5.png">

We want our Lamda function to be triggered by our Alexa skill, so go ahead and click the Alexa Skills Kit option.
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/6.png">

Now we get to name our function (highRoller is good enough) and upload our src folder (node modules and all) in a zip file. Zip up your src, and send it up!
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/7.png">

You might notice that my zip file here is 158 bytes. This actually due to hyper-efficient compression algorithms of my own design, and not because I forgot to actually include any files. Yours should be bigger.

Finally, we need to make a function role for our Lamda funtion. Using the templates, select "Simple Microservice Permissions" and name your role something like "basic_execution". You can leave the handler alone.
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/8.png">

Click create, aaaaand... we're done! Copy the ARN in the upper right, and now we can wire the skill itself up to our interaction model.
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/9.png">

Head back to the developer site, choose Lambda as your resource, check North America, and paste your ARN in.
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/10.png">

That's it! Now we can start testing our skill!
<img src="https://github.com/jcquery/alexa-sdk-lesson/raw/master/screenshots/11.png">

Further reference:

[Alexa Node SDK Repo](https://github.com/alexa/alexa-skills-kit-sdk-for-nodejs)

[Alexa tutorials](https://developer.amazon.com/alexa-skills-kit/alexa-skills-developer-training)

[Custom Skill Docs](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/overviews/understanding-custom-skills)
