# SMS Two-Factor Authentication with Node.js

## Introduction

Traditionally, authentication systems have relied solely on usernames and passwords. As multiple security breaches have shown, these systems are often not enough to protect users, who often use only weak passwords. Implementations of two-factor authentication (2FA) provide an additional layer of account security by augmenting the user's password with a second authentication token, and they have become increasingly popular. Common two-factor authentication methods typically use one-time passwords (OTP), either generated by hardware tokens or authenticator apps or directly sent to the user's mobile phone via SMS text messaging. SMS-based two-factor authentication has the advantage of not requiring additional software or hardware on the user's end. With APIs - such as MessageBird's Verify API - readily available, they are also straightforward to implement for developers.

In this tutorial, we'll introduce you to MessageBird's Verify API and build a minimal implementation of the authentication process in NodeJS. This example has the following steps:

- Asking for the phone number
- Sending a verification code
- Verifying the code

You can follow along the tutorial to build the whole sample application from scratch or, if you want to see it in action right away, you can also [download, clone or fork the sample repository from GitHub](https://github.com/).

## Prerequisites

In the course of this tutorial, we'll build an application using Javascript. We'll use [Node.js](https://nodejs.org/en/), the [Express framework](https://github.com/expressjs/express) and the [Handlebars templating engine](https://www.npmjs.com/package/express-handlebars) as well as the [MessageBird SDK](https://www.npmjs.com/package/messagebird). As we'll explain every step along the way, you should be able to follow the tutorial even with none or limited knowledge of Express and Handlebars.

Node's package manager - npm -should already be installed on your computer. If it isn't, [you can download npm here](https://www.npmjs.com/get-npm).

## Project Setup

### Dependencies

Create a new directory on your computer to store the sample application and create a file called `package.json` inside the directory with the following content:

````json
{
  "name": "node-messagebird-verify-example",
  "main": "index.js",
  "dependencies": {
    "express": "^4.16.2",
    "express-handlebars": "^3.0.0",
    "body-parser": "^1.18.2",
    "messagebird": "^2.1.4"
  }
}
````

This file provides a name for our sample application, declares the main file, which we'll create next, and lists the dependencies we need along with their versions. Apart from Express, Handlebars, and the MessageBird SDK, which we've already mentioned above, we're adding a small helper library called body parser to simplify our code.

Open a console pointed to the directory that contains the file you just created and type the following command to have npm install all required modules into your project:

````bash
npm install
````

### Main file

Create an `index.js` file in the same directory as your `package.json` next. This file starts off by including the dependencies:

````javascript
var express = require('express');
var exphbs  = require('express-handlebars');
var bodyParser = require('body-parser');
````

For including the MessageBird SDK we need to provide an access key for the API. MessageBird provides keys in _live_ and _test_ modes. For this tutorial you should use a live key. Otherwise, you will not be able to test the complete flow.

Go to the [MessageBird Dashboard](https://dashboard.messagebird.com/en/user/index); if you have already created an API key it will be shown right there. Click on the eye icon to make the access key visible, then select and copy it to your clipboard. If you do not see any key on the dashboard or if you're unsure whether this key is in _live_ mode, go to the _Developers_ section and open the [API access (REST) tab](https://dashboard.messagebird.com/en/developers/access). Here, you can create new keys and manage your existing ones.

Append the following line to your `index.js`, replacing the string YOUR_API_KEY with a live access key from your account:

````javascript
var messagebird = require('messagebird')('YOUR_API_KEY');
````

Next, we initialize and configure the framework and enable Handlebars and the body parser:

````javascript
var app = express();
app.engine('handlebars', exphbs({defaultLayout: 'main'}));
app.set('view engine', 'handlebars');
app.use(bodyParser.urlencoded({ extended : true }));
````

### Views

We use Handlebars to separate the logic of our code from the HTML pages. Create a directory named `views`. Inside `views`, create another directory named `layouts` and a file called `main.handlebars` inside with the following content:

````html
<!doctype html>
<html>
    <head>
        <title>MessageBird Verify Example</title>
    </head>
    <body>
        <h1>MessageBird Verify Example</h1>
        
        {{{body}}}
    </body>
</html>
````

This is the main layout which acts as a container for all pages of our application. We'll create the views for each page next.

## Asking for the phone number

The obvious first step in verifying a user's phone number is asking them for that number. Create an HTML form for this purpose and store it as `step1.handlebars` inside the `views` directory:

````html
{{#if error}}
    <p>{{error}}</p>
{{/if}}
<p>Please enter your phone number (in international format, starting with +) to receive a verification code:</p>
<form method="post" action="/step2">
    <input type="tel" name="number" />
    <input type="submit" value="Send code" />
</form>
````

The form is simple, having just one input field and one submit button. Providing `tel` as the `type` attribute of our input field allows some browsers, especially on mobile devices, to optimize for telephone number input, for example by displaying a numberpad-style keyboard. The section starting with `{{#if error}` is needed to display errors. We'll get back to it in a minute.

Add a route to your `index.js` to display the page:

````javascript
app.get('/', function(req, res) {
    res.render('step1');
});
````

## Sending a verification code

Once we've collected the number, we can send a verification message to the user's mobile device. MessageBird's Verify API takes care of generating a random token, so you don't have to do this yourself. Codes are numeric and six digits by default, which is suitable for our demo. You can refer [to the Verify API documentation](https://developers.messagebird.com/docs/verify#verify-request) if you want to change the length of the code or configure other options.

The form we created in the last step submits the phone number via HTTP POST to `/step2`, so let's define this route in our `index.js`:

````javascript
app.post('/step2', function(req, res) {
    var number = req.body.number;    
    messagebird.verify.create(number, {
        template : 'Your verification code is %token.'
    }, function (err, response) {
        if (err) {
            console.log(err);
            res.render('step1', {
                error : err.errors[0].description
            });
        } else {
            console.log(response);
            res.render('step2', {
                id : response.id
            });
        }
    })    
});
````

Let's dive into what happens here: First, the number is taken from the request. Then, we're calling `messagebird.verify.create()` with the number as the first parameter. The second parameter is used to provide additional options; in our case, we're using a template to phrase the wording of the message. The template contains the placeholder `%token`, which is replaced with the generated token on MessageBird's end. If we omitted this, the message would simply contain the token and nothing else.

Like most JavaScript functions, the MessageBird API call is asynchronous and transfers control to a callback function once finished. In the SDK's case, this callback function takes two parameters, `err` and `response`.

If `err` is set, it means that an error has occurred. A typical error could be, for example, that the user has entered an invalid phone number. For our demo, we merely re-render the page from our first step and pass the description of the error into the template - remember the `{{#if error}` section from the first step?! In production applications, you'd most likely not expose the raw API error. Instead, you could consider different possible problems and return an appropriate message in your own words. You might also want to prevent some errors from happening by doing some input validation on the phone number yourself. We've also added a `console.log(err)` statement so you can explore the complete error object in your console and learn more about how it works.

In case the request was successful, we'll render a new page. Our API response contains an ID, which we'll need for the next step, so we'll just add it to the form. Since the ID is meaningless without your API access key there are no security implications of doing so, however, in practice, you'd be more likely to store this ID in a session object on the server. Just as before, we're logging the whole response to the console for debugging purposes. We still need to build the new page, so create a file called `step2.handlebars` in your `views` directory:

````html
{{#if error}}
    <p>{{error}}</p>
{{/if}}
<p>We have sent you a verification code!</p>
<p>Please enter the code here:</p>
<form method="post" action="/step3">
    <input type="hidden" name="id" value="{{id}}" />
    <input type="text" name="token" />
    <input type="submit" value="Check code" />
</form>
````

The form is very similar to the first step. Note that we include a hidden field with our verification ID and, once again, have a conditional error section.

## Verifying the code

The user will check their phone and enter the code into our form. What we need to do next is send the user's input along with the ID of the verification request to MessageBird's API and see whether the verification was successful or not. Let's declare this third step as a new route in our `index.js`:

````javascript
app.post('/step3', function(req, res) {
    var id = req.body.id;
    var token = req.body.token;
    messagebird.verify.verify(id, token, function(err, response) {
        if (err) {
            console.log(err);
            res.render('step2', {
                error: err.errors[0].description,
                id : id
            });
        } else {
            console.log(response);
            res.render('step3');
        }
    })    
});
````

This code looks very similar to the one in the second step. First, we're reading the input and then make a call to MessageBird's API. This time, it's the `messagebird.verify.verify()` method, which accepts `id` and `token` as its parameters. Inside the callback, error and success cases are handled.

In case of an error, such as an invalid or expired token, we're showing that error on our page from the second step.

In the success case, we simply show a new page. Create this page in your `views` directory and call it `step3.handlebars`:

````html
<p>You have successfully verified your phone number.</p>
````

## Testing

One final line is missing in your `index.js`, the one which actually runs the Express application you built:

````javascript
app.listen(8080);
````

Now, take a quick look at the directory structure you created. It should look something like this:

````
node_modules
views
- layouts
- - main.handlebars
- step1.handlebars
- step2.handlebars
- step3.handlebars
index.js
package-lock.json
package.json
````

If anything is missing double-check if you've executed all steps in the tutorial successfully.

If you're all set, save your `index.js` and run the application from the command line:

````
node index.js
````

Point your browser to [http://localhost:8080/](http://localhost:8080/) and try to verify your own phone number.

## Final Remarks

Congratulations, you have completed the tutorial and now have a running integration of MessageBird's Verify API!

You can now leverage the flow, code snippets and UI examples from this tutorial to build the verification into a real application's register and login process to enable 2FA for it.

Is it not working for you? Have a look at [the complete code on GitHub](https://github.com/) to see whether you might have missed something. Thanks for your interest in MessageBird's tutorials! Feel free to browse around for more guides.