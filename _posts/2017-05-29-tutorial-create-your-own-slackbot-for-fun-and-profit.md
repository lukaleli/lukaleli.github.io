---
layout: post
title:  "TUTORIAL: Create your own Slackbot for fun and profit"
date:   2017-05-29 14:34:25
categories: slack slackbot javascript nodejs
image: /assets/article_images/2017-05-29-slackbot/slackbot-cover-dark.jpg
---

I work at [Wertewandel](https://www.wertewandel.de/) - a startup where we build a bonus programme for sustainable products. Recently our backend developer exposed an API endpoint for retrieving a number of customers that already signed up for our programme. I thought this is a great opportunity to create a simple Slackbot as Slack is our tool of choice for communication purposes.

---

## Requirements  

1. We want to be notified about the current number of customers on workdays during our working hours, e.g. from Monday to Friday at 12 a.m.
2. If someone's too impatient to wait for official Slackbot announcement one can ask bot directly via Direct Message on Slack

That should be sufficient to get started. 


*This tutorial is pretty basic and I'll try to explain as much code as possible.*

---

## Code

I'm gonna use [Node](https://nodejs.org/en/) and Javascript to write our Slackbot. 

First we need to create a directory for our project and initialize it via `npm` to manage our dependencies. I assume you have Node installed in your system (I use v7.7.3 to use some of ES6 syntax goodies).

{% highlight bash %}

mkdir slackbot
cd slackbot
touch app.js
npm init -y

{% endhighlight %}

Great! We have a basic directory structure to start writing the code.

---

### Talking with Slack

Slack has a Real Time Messaging API to send messages back and forth to Slack via websockets. You can find a detailed documentation [here](https://api.slack.com/rtm). We could, of course, use that documentation to connect our app to the Slack's API and do everything by hand, but we, programmers, are lazy by nature and if we can take some shortcuts we are usually happy to do so. We're gonna use a third party library that serves as a convenient wrapper around RTM Slack API and exposes methods to easily connect to the Slack API and operate on it. I've searched through Github and decided to use [Slackbots.js](https://github.com/mishk0/slack-bot-api) library.

Let's install it and save as a project's dependency

{% highlight bash %}

npm i -S slackbots

{% endhighlight %}

But before we start writing actual code let's take a look at the Slackbots.js library documentation.
 [Usage section](https://github.com/mishk0/slack-bot-api#usage) shows how to create SlackBot instance. 

**app.js**
{% highlight js %}

var SlackBot = require('slackbots');

// create a bot
var bot = new SlackBot({
    token: 'xoxb-012345678-ABC1DFG2HIJ3', // Add a bot https://my.slack.com/services/new/bot and put the token 
    name: 'My Bot'
});

{% endhighlight %}

To gain access to our team's Slack we need to generate access token by visiting [https://my.slack.com/services/new/bot](https://my.slack.com/services/new/bot). 

You'll see a screen to create a new bot integration

![](/assets/article_images/2017-05-29-slackbot/slack-api-screenshot.png)

Choose a name for your bot and click **Add bot integration** button which will redirect you to the page with generated token. That's all what we need. Copy the token string and save it somewhere in the secure place. We don't want anybody to have access to our Slack team and listen to our super secret product strategies 😉

Now let's open our **app.js** file and initialize Slackbot with two parameters: our access token and slackbot's name (it doesn't have to be the same name that you specified in the bot integration panel).

**app.js**
{% highlight js %}

const SlackBot = require('slackbots')

const TOKEN = process.env.BOT_API_TOKEN
const BOT_NAME = 'Slackbot'

const bot = new SlackBot({
    token: TOKEN,
    name: BOT_NAME
})

{% endhighlight %}

I've declared two constants `TOKEN` and `BOT_NAME` for clarity. As you can see `TOKEN` constant has assigned value residing in the Node environmental variable called `BOT_API_TOKEN`. Why is that? Simply put: it's a good practice to inject secret values to our applications via env variables rather than keeping them in version control system like Git. Having that you can securely keep your code in a public repository without worrying that someone can use your credentials for some nasty purposes. 

For safety sake we can even throw an error in case you forgot to specify it when starting a Node process.

**app.js**
{% highlight js %}

const SlackBot = require('slackbots')

const TOKEN = process.env.BOT_API_TOKEN
const BOT_NAME = 'Slackbot'

if (!TOKEN) {
    throw new Error('BOT_API_TOKEN not specified')
}

const bot = new SlackBot({
    token: TOKEN,
    name: BOT_NAME
})

{% endhighlight %}

Now let's test out if we can successfully send a message to one of our Slack channels.

**app.js**
{% highlight js %}

const SlackBot = require('slackbots')

const TOKEN = process.env.BOT_API_TOKEN
const BOT_NAME = 'Slackbot'
const CHANNEL = 'general'
const PARAMS = {
    icon_emoji: ':heart:'
}

if (!TOKEN) {
    throw Error('BOT_API_TOKEN not specified')
}

const bot = new SlackBot({
    token: TOKEN,
    name: BOT_NAME
})

bot.on('start', () => {
    bot.postMessageToChannel(CHANNEL, 'It\'s alive!!!', PARAMS)
})

{% endhighlight %}

Let's stop here for a moment and see what happens in the code. We've added new constant variables: `CHANNEL` and `PARAMS`. `CHANNEL` is simply a name of the channel you want to post your message to. `PARAMS` is the parameters object you can pass to `postMessageToChannel` method to specify e.g. emoji that's gonna be used as an avatar for our Slackbot.
Now switch to the terminal and type

{% highlight bash %}

$ BOT_API_TOKEN=<YOUR_SECRET_TOKEN> node app.js

{% endhighlight %}

in your project's directory. 

**Don't forget to replace `<YOUR_SECRET_TOKEN>` with your secret token!**

If you followed the instructions carefully you should receive **It's alive!!!** message with ❤️ as an avatar at your team's *#general* Slack channel. Hurray!

Our bot is still listening for the messages from the Slack API and the node process is still running. We can terminate it by pressing `Ctrl + C` in the terminal.

---

### Showing real data

By now we do know how to send a message, but it's not really useful. Let's try to get a customers number from the endpoint I've mentioned at the beginning of this post. Before we write the code we can check if it works correctly and returns a value. I'm gonna make a **GET** request with cURL in the terminal to test our endpoint.

*NOTE: To be precise: I'm not using cURL, but [HTTPie](https://httpie.org/) which is a nice utility tool that provides more user-friendly commands to make HTTP requests than cURL itself. I always forget how to make more complex requests with cURL and HTTPie helps a lot :)*

*NOTE 2: I'm using a random number generator API instead of my company API endpoint. Numbers are not real, but are still numbers that we can use for the sake of this tutorial :)*

{% highlight bash %}

http -v 'https://www.random.org/integers/?num=1&min=1&max=500&col=1&base=10&format=plain&rnd=new'
GET /integers/?num=1&min=1&max=500&col=1&base=10&format=plain&rnd=new HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Host: www.random.org
User-Agent: HTTPie/0.9.8



HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
CF-RAY: 3656d75d9e2f2af7-WAW
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Connection: keep-alive
Content-Encoding: gzip
Content-Length: 24
Content-Type: text/plain;charset=utf-8
Date: Sat, 27 May 2017 06:15:43 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Pragma: no-cache
RDO-Authenticated-Login: -
Server: cloudflare-nginx
Set-Cookie: __cfduid=d778c33106e2ea51e6d91a114cb7d7ae71495865743; expires=Sun, 27-May-18 06:15:43 GMT; path=/; domain=.random.org; HttpOnly
Set-Cookie: RDOSESSION=0v32nd8kg0jme2hho5t0boeg56; path=/; domain=.random.org
Vary: Accept-Encoding
X-Powered-By: PHP/5.4.45-0+deb7u8

280

{% endhighlight %}

Looks like we already have 280 customers and that means the endpoint is working correctly. Let's try to fetch that number in our application. I'll use [node-fetch](https://github.com/bitinn/node-fetch) module that implements `window.fetch` method compatible with its browser's counterpart. If you're not familiar with the fetch API I suggest you read [David's Walsh great introduction post](https://davidwalsh.name/fetch) first. We will implement that functionality as a separate module named **fetcher.js**.


**fetcher.js**

{% highlight js %}

const fetch = require('node-fetch')
const TRACKER_URL = 'https://www.random.org/integers/?num=1&min=1&max=500&col=1&base=10&format=plain&rnd=new'

const fetchCount = () => {
    return fetch(TRACKER_URL)
        .then(response => response.text())
        .then(response => parseInt(response))
}

module.exports = fetchCount

{% endhighlight %}

Our **fetcher.js** module exports `fetchCount` method that returns promise. But before passing the result further into promise chain our method runs two transformations on the fetched data: 

- `text()` method (which returns promise) on the `Response` object to convert response body to the plain text (if we would expect our response to be in the JSON format we would use `json()` method instead),

- `parseInt` to convert our string to the integer.

Now let's try to use our module to fetch data from the server and send to our Slack channel.

**app.js**
{% highlight js %}

const SlackBot = require('slackbots')
const fetchCount = require('./fetcher')

const TOKEN = process.env.BOT_API_TOKEN
const BOT_NAME = 'Slackbot'
const CHANNEL = 'general'
const PARAMS = {
    icon_emoji: ':heart:'
}

if (!TOKEN) {
    throw Error('BOT_API_TOKEN not specified')
}

const bot = new SlackBot({
    token: TOKEN,
    name: BOT_NAME
})

bot.on('start', () => {
    fetchCount()
        .then(count => {
            bot.postMessageToChannel(CHANNEL, 'Yupi! We have ' + count + ' customers!', PARAMS)
        })
        .catch(error => {
            bot.postMessageToChannel(CHANNEL, 'Houston, we have a problem with customers tracker!', PARAMS)
        })
})

{% endhighlight %}

We basically wait for `fetchCount` promise until it's resolved and then we use passed customers number value to create Slack message and send it. Sometimes an error can occur that's why we also add `catch` at the end to catch any unexpected errors and notify users on Slack that something went wrong.

---

### Scheduling the task

Now we need to take care of sending messages at a scheduled time- in our case on Mon-Fri at 12 a.m. The first thing that comes to my mind when I think about scheduling is **cron**. Quick search for *"node cron"* phrase on Google shows a [github.com/kelektiv/node-cron](https://github.com/kelektiv/node-cron) which we will use for our project. **Cron** tool uses special pattern passed as a string to reflect times at which specified task will be run. You can read more about these configuration patterns [here](http://crontab.org/). I've prepared a following pattern


`00 00 12 * * 1-5`

which, in cron's language, means *"run task at 12 every day from Monday till Friday"* as required. After a short glance at *node-cron* documentation we're ready to add a task to our application.

**app.js**
{% highlight js %}

const CronJob = require('cron').CronJob
const SlackBot = require('slackbots')
const fetchCount = require('./fetcher')

const TOKEN = process.env.BOT_API_TOKEN
const BOT_NAME = 'Slackbot'
const CHANNEL = 'general'
const PARAMS = {
    icon_emoji: ':heart:'
}
const CRON_PATTERN = '00 00 12 * * 1-5'
const TIME_ZONE = 'Europe/Berlin'

if (!TOKEN) {
    throw Error('BOT_API_TOKEN not specified')
}

const bot = new SlackBot({
    token: TOKEN,
    name: BOT_NAME
})

const task = () => {
    fetchCount()
        .then(count => {
            bot.postMessageToChannel(CHANNEL, 'Yupi! We have ' + count + ' customers!', PARAMS)
        })
        .catch(error => {
            bot.postMessageToChannel(CHANNEL, 'Houston, we have a problem with customers tracker!', PARAMS)
        })
}

const onCronError = error => {
    console.error('Cron job stopped', error)
}

bot.on('start', () => {
	new CronJob(CRON_PATTERN, task, onCronError, true, TIME_ZONE)
})

{% endhighlight %}

I've refactored `app.js` code a bit and added a cron job. Please note that I've used Berlin's timezone to which cron will refer when running it's jobs. Please change it according to your time zone.

We're almost there, but there's still a second requirement we didn't cover: our slackbot needs to respond to direct messages. To do that I needed to dig a bit deeper into documentation to filter out all the incoming data that's not relevant to us, because when we listen for `message` event we will be notified on almost every event happening on our team's Slack. 

**app.js**
{% highlight js %}

const CronJob = require('cron').CronJob
const SlackBot = require('slackbots')
const fetchCount = require('./fetcher')

const TOKEN = process.env.BOT_API_TOKEN
const BOT_NAME = 'Slackbot'
const CHANNEL = 'general'
const PARAMS = {
    icon_emoji: ':heart:'
}
const CRON_PATTERN = '00 00 12 * * 1-5'
const TIME_ZONE = 'Europe/Berlin'

if (!TOKEN) {
    throw Error('BOT_API_TOKEN not specified')
}

const bot = new SlackBot({
    token: TOKEN,
    name: BOT_NAME
})

const task = () => {
    fetchCount()
        .then(count => {
            bot.postMessageToChannel(CHANNEL, 'Yupi! We have ' + count + ' customers!', PARAMS)
        })
        .catch(error => {
            bot.postMessageToChannel(CHANNEL, 'Houston, we have a problem with customers tracker!', PARAMS)
        })
}

const onCronError = error => {
    console.error('Cron job stopped', error)
}

const onDirectMessage = channel => {
    fetchCount()
        .then(count => {
            bot.postMessage(channel, 'Oh,geez! You're so impatient! We have *' + count + ' customers*. Happy now? Bye bye. :angry:', PARAMS)
        })
        .catch(error => {
            bot.postMessage(channel, 'Something is not working properly. I couldn't get a customers number.', PARAMS)
        })
}

bot.on('start', () => {
    new CronJob(CRON_PATTERN, task, onCronError, true, TIME_ZONE)
})

bot.on('message', data => {
    if (data.type === 'message' && data.bot_id === undefined) {
        bot.openIm(data.user).then(im => {
            if (im.channel.id === data.channel) {
                onDirectMessage(data.channel)
            }
        })
    }
})

{% endhighlight %}

Our feature to handle direct messages is a bit stupid now because it will respond with customers number to any random string you send to him. It would be great if our bot could respond meaningfully to our questions. But I'll leave these improvements for you as a homework 🙂

I hope you've tested the code and confirmed that it works on your local machine. Of course this is not how we're gonna leave it. In the next section I'll show you how to deploy your application to the server. For free. 🙌

---

### Make it alive

There are tons of cloud options to deploy your app to, but I want to make it as painless as possible and, first of all, free. I've chosen [Heroku](https://www.heroku.com/) for that purpose as a wide-known, developer-friendly and, to some point, free application hosting service.

But before we configure Heroku I want you to create a repository on your Github account to keep Slackbot code in it. Aside from being a good practice to version control your code we'll use it for an easy deployment to Heroku.

When you have your Github repository all set let's go back to the Heroku dashboard and click **New** -> **Create new app** button under **Personal apps** section

![](/assets/article_images/2017-05-29-slackbot/step_1.png)

In **Create New App** section provide unique name for your app or just leave it blank and let Heroku do that for you by generating a random string. You can also choose datacenter location: either Europe or United States. Choose the one that's closer to your location to save some time on the packets delivery 😉

![](/assets/article_images/2017-05-29-slackbot/step_2.png)

After you create the app go to the **Deploy** bookmark.

![](/assets/article_images/2017-05-29-slackbot/step_3.png)

Click the **Github** button in the **Deployment method** section, authorize Heroku app on your Github account and then type the name of your repo that you created for your Slackbot app. Then click the **Search** button and your repository should be listed below. Click **Connect**. 

![](/assets/article_images/2017-05-29-slackbot/step_5.png)

After your repository connected successfully scroll down to the **Automatic deploys** section and click **Enable Automatic Deploys** button. Right now whenever you commit new changes to your Slackbot's repository the app will be redeployed with your changes. Neat, huh? 

![](/assets/article_images/2017-05-29-slackbot/step_6.png)

Now switch to the **Settings** bookmark and click **Reveal Config Vars**.

![](/assets/article_images/2017-05-29-slackbot/step_7.png)

Do you remember our environmental variable `BOT_API_TOKEN`? This is the place where we can set any env variable so we don't have to set it manually when starting our app.

![](/assets/article_images/2017-05-29-slackbot/step_8.png)

Now go back to your terminal, `cd` you project's directory and execute a following command

{% highlight bash %}

echo "worker: node app.js" > Procfile

{% endhighlight %}

This command will create *Procfile* file that specifies the type of the app we're deploying and a command to start the app. *Procfile* is a Heroku specific configuration file. 

When you've done that commit changes and push to your repository.

That should trigger deployment process on Heroku which you can observe in **Activity** tab. After deployment is finished go to the **Resources** tab, turn off *web* dyno type and turn on *worker* (you can do this by clicking pencil button and toggling the switch button). You should have it configured as in the picture below.

![](/assets/article_images/2017-05-29-slackbot/step_9.png)

The difference between *web* and *worker* process type is that *web* simply runs a web server that exposes our application to the outside world with UI and *worker* is simply a Node process that doesn't need to serve anything to the end user.

Now go back to **Deploy** bookmark, scroll down to the very bottom and click **Deploy branch** with *master* branch chosen. That will cause redeploy with updated dyno configuration.

You can check server logs (**More** -> **View logs**) to see if everything went fine and our app didn't crash. 

![](/assets/article_images/2017-05-29-slackbot/step_10.png)

**Note:** I like to add various `console.log` statements in the code to see if my services initialized successfully (e.g. on bot start event) and `console.error` when something has failed. You should do the same to easily track problems with your code on production.

If there's no suspicious messages in the logs and you can see **Build succeeded** green message in the **Activity** tab on your latest deployment

![](/assets/article_images/2017-05-29-slackbot/step_11.png)

and when you send a direct message to your Slackbot in your Slack app and you see this in the response

![](/assets/article_images/2017-05-29-slackbot/slack_msg.png)

then **congrats**!!! You've just created your first Slackbot!

---

### Make it fun

Our app is super helpful in informing our team about how we're doing. But these messages are boring, always the same. What if we could enhance our code to randomize the messages and do a bit of motivational speeches like american army officers do? 😉 

Let's create another module called **messages.js** that will take the current number of customers, compare it with previous one and based on that return an appropriate message (either harsh and motivational or with congrats and love).

**messages.js**
{% highlight js %}

Array.prototype.random = function() {
  return this[Math.floor(Math.random() * this.length)]
}

String.prototype.format = function(...args) {
  return this.replace(/{(\d+)}/g, function(match, number) {
    return typeof args[number] !== 'undefined' ? args[number] : match
  })
}

const sameCountMessages = [
  "Hello, fellows! Looks like we don't work too hard because we still have only *{0} customers*! Nothing's changed from the last time! Chop, chop! Work harder! :v:",
  "Goddammit! We still have only *{0} customers*. I promise you, I'm gonna stick this pace stick through your ears and ride you round this parade square like a shagging motorbike! Get to work!! :angry:",
  'Guys, did u ever pick up your teeth with broken fingers? You will learn soon. *{0} bloody customers*',
  'Hi, guys... I feel like I am the only one doing the real work here... We have only *{0} people* onboard!!! Come on! I believe that you can do better! :rick:',
  "What's wrong with you guys?! Still *{0} customers*? No progress? I’m going to rip your arm off and slap you with the soggy end if there's no improvement soon!",
  "Hey guys! Who's the master? Huh? Who's the master? Certainly not we! Only *{0} customers*? We certainly can do better! Up up up! Get to work!",
  "Only *{0} customers*... I'm going to hurt you, guys, *alot*, and *slowly*...",
  "This is the last time I'm coming here and there's no improvement! We have poor number of *{0} customers*... It's laughable! I'm gonna make you my sex slaves next time if this doesn't change! Bye!",
]

const decreasedCountMessages = [
  'WTF PEOPLE?! *{0} customers*??!! Even my mother can do better!! NOT SAYING BYE THIS TIME! :point_up:',
  'Guys, seriously! I am going to skin you and use it as a fckng wet suit this weekend. Customers are leaving us! Only *{0}* left! Aaaaaaah!',
  ":middle_finger: We're losing them!!! Only *{0}* left!!! Aaaarrggghhhhhh, I need to get a drink... :middle_finger: :middle_finger: :middle_finger:",
  "Noooooooo! That's it! I'm done! Fuck it fuck it fuck it! Only *{0} customers* left. It means that tomorrow we can all go home! :rage:",
]

const increasedCountMessages = [
  'Hello! I want to proudly announce that we are badass! *{0} customers* and counting! Love you guys, keep up the good work! :kiss:',
  ":raised_hands: Getting down on my knees... You're awesome guys! We've reached *{0} customers*! Tequilla shots on me! Keep it up- don't you ever stop! :pray:",
  "What a beeeeeeeaaaaaauuuuuutiiiiiiifuuuuuuul day! We reached enormous number of *{0} customers* in our garden, which means we're getting there! Love you guys! :heart:",
  "Uh la la! Who's the master? Huh? Who's the master? We are! Next customers arriving slowly but steady! *{0} people* on board! Yupiii! :muscle:",
]

const messages = [
  { cond: (prev, current) => prev < current, msg: increasedCountMessages },
  { cond: (prev, current) => prev > current, msg: decreasedCountMessages },
  { cond: (prev, current) => prev === current, msg: sameCountMessages },
]

const getMessage = (prev, current) => {
  for (const i in messages) {
    if (messages[i].cond(prev, current)) {
      return messages[i].msg.random().format(current)
    }
  }
}

module.exports = getMessage

{% endhighlight %}

And in our **app.js**

**app.js**
{% highlight js %}

const SlackBot = require('slackbots')
const CronJob = require('cron').CronJob
const fetchCount = require('./fetcher')
const getMessage = require('./messages')

const TOKEN = process.env.BOT_API_TOKEN
const BOT_NAME = 'Funny Slackbot'
const CHANNEL = 'general'
const PARAMS = {
  icon_emoji: ':heart:',
}
const CRON_PATTERN = '00 00 12 * * 1-5'
const TIME_ZONE = 'Europe/Berlin'

let previousCount = 0

if (!TOKEN) {
  throw new Error('BOT_API_TOKEN not specified')
}

const bot = new SlackBot({
  token: TOKEN,
  name: BOT_NAME,
})

const task = () => {
  console.log('--- TASK STARTED ---')
  fetchCount()
    .then(count => {
      bot.postMessageToChannel(
        CHANNEL,
        getMessage(previousCount, count),
        PARAMS
      )
      previousCount = count
    })
    .catch(error => {
      bot.postMessageToChannel(
        CHANNEL,
        'Houston, we have a problem with customers tracker!',
        PARAMS
      )
    })
}

const onCronError = error => {
  console.error('Cron job stopped', error)
}

const onDirectMessage = channel => {
  fetchCount()
    .then(count => {
      bot.postMessage(
        channel,
        "Oh,geez! You're so impatient! We have " + count + " customers*. :angry:",
        PARAMS
      )
    })
    .catch(error => {
      bot.postMessage(
        channel,
        "Something is not working properly. I couldn't get a customers number.",
        PARAMS
      )
    })
}

bot.on('start', () => {
  console.log('--- BOT STARTED ---')
  new CronJob(CRON_PATTERN, task, onCronError, true, TIME_ZONE)
})

bot.on('message', function(data) {
  if (data.type === 'message' && data.bot_id === undefined) {
    bot.openIm(data.user).then(im => {
      if (im.channel.id === data.channel) {
        onDirectMessage(data.channel)
      }
    })
  }
})

{% endhighlight %}

In **messages.js** I've added two prototype methods to the `Array` and `String`:

- `random()` for convenient random picking the values from the array

- `format()` for easy value injection into placeholders specified in the string

The `messages` array serves as a structure to keep our rules for picking up the message from the appropriate array according to the `prev` and `next` customers number value.

Then our `getMessage()` method randomly picks the string from the array, injects the current customers number and returns it.

In **app.js** we've added `previousCount` variable to keep the last number from our API for comparison purposes.

That'd be all! Thanks for reading. I hope you liked it. 

Until next time! 


---

The source code is available at [https://github.com/lukaleli/slackbot](https://github.com/lukaleli/slackbot)



