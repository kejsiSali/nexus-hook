# Blackout Nexus hooks

![Blackout logo](https://blackout.ai/img/logo/logo.png)

|Author|Email|Latest version|State|
|---|---|---|---|
|Marc Fiedler|dev@blackout.ai|0.4.0|`BETA`

## License
|Copyrite|License
|---|---|
|Blackout Technologies|GPLv3|


Hooks are custom scripts that can be integrated into any point within a dialog
to access external or living data. Any information and data that is not part of
the original A.I. training falls under the category `living data`.

## Hook integration

In order to develop a hook you first need to subclass the `Hook` class from this
library. You need to implement the function `process()` in order to be able to
handle data from the dialog engine.

### Prepare training function

During the *training phase* of the artificial intelligence, you will be able to hook
yourself into the process of preparing training data.

During training preprerations `prepareTraining` is called with the following parameters:

|Parameter|Type|Description|
|---|---|---|
|phrases|Object|This is the intent that triggered the hook. this object has a `name` and a `confidence` field.|
|complete|Callback|This is the completion function, complete has to be called with an array parameter. The array must be a list of objects that have the following layout: `{id: <phraseID>, text: <phraseText>}`|

```JavaScript
/**
 *  @param {String} intent Name of the intent that is currently in use
 *  @param {Array} phrases Array that contains all phrasing examples for the intent that triggers this Hook.
 *  @param {Callback} complete Completion callback to continue dialog
 */
prepareTraining(intent, phrases, complete){
    var samples [];

    // go through all phrasings and find the once that we want
    // to manipulate
    for( var i in phrases ){
        if( phrases[i].indexOf("$city") > -1 ){
            samples.push(phrases[i]);
        }
    }

    // return the samples back to the training service
    complete(samples);
}
```

### Process function

The `process()` function is the heart of the hook, it has the following parameters:

|Parameter|Type|Description|
|---|---|---|
|intent|Object|This is the intent that triggered the hook. this object has a `name` and a `confidence` field.|
|text|String|This string contains the original phrasing of the user|
|session|Object|The complete user session, including the message thread|
|complete|Callback|This is the completion function, it has to be called at the end of your script in order to continue dialog.|

 *Additional Variables*
 `this.json` contains the POST request that the user client sent to the chat bot.

### Completion callback

The completion callback needs to be called at the end of what your
hook is doing. You have to transport at least a `answer` string through
the completion callback. Optionally you can also add a `platform` object
that can hold additional platform information like buttons, images, actions.
Example:

```JavaScript
complete({
    answer: "I love cookies!",
    platform: {
        buttons: [
            {caption: "Give more cookies", action: "give more cookies"},
            {caption: "Eat all cookies yourself", action: "i eat all cookies myself"}
        ]
    }
})
```

### Simple Eample implementation

```JavaScript
// load the Hook class from this library
const Hook = require('nexus-hook').Hook;

// create your own hook as subclass of Hook
module.exports = class WeatherHook extends Hook {
    /**
     *  @param {Object} intent Object with .name and .confidence
     *  @param {String} text Original phrasing of the user
     *  @param {Callback} complete Completion callback to continue dialog
     */
    process(intent, text, complete){
        // since we already know the phrasing, we can ignore the
        // intent and text parameter
        this.request('GET', 'http://api.openweathermap.org/data/2.5/weather?q=Bremen&units=metric&appid=<ID>', {}, (resp) => {
            // after the response from the request-pronise came back
            // complete this hook with an answer string and optionally
            // with a platform object.
            complete({
                answer: 'The weather in Bremen is: '+resp.weather[0].main+" with "+resp.main.temp+" degrees.",
                platform: {}
            });
        });
    }
}
```

# Handling languages
If you want to provide multi-lingual support to your hook you can do that with the `languageDict` implementation that comes with the `nexus-hook` library.

Each hook has a member variable called `captions` you can access it via `this.captions`. You can get a language specific phrasing with the `get()` function of the `captions` object.

In order to have a working language dictionary, you need to create a file called `languageDict.json` in the root folder of your hook.

## Example languageDict file

```json
{
    "de": {
        "fallback": "Ich habe keine informationen zu deiner Suche gefunden",
        "weatherAnswer": "Das Wetter in $city ist $weather mit $temperature grad"
    },
    "en": {
        "fallback": "Sorry, I can't find anything.",
        "weatherAnswer": "The weather in $city is $weather with $temperature degrees"
    }
}
```

If the setup of the `languageDict.json` was done correctly, you will be able to access any caption with the `get()` function passing the name of the caption as a string.

## Example usage
```JavaScript
this.captions.get('fallback'); // in the above example will return:  "Sorry, I can't find anything." if the language is english
```

# Testing
The integrations in the `nexusUi` as of version <= *2.0.0* will not support error feedback. In order to test your hook, you can utilize the `TestHook` class of the `nexus-hook` library.

The `TestHook` class has only one function called `chat` the prototype of the chat function looks like this:

```JavaScript
chat(intent, text, complete)
```

You will have to pass an intent name and a text to the chat function of the `TestHook` class. The response will then be returned through the complete callback.

The rest of the hook will behave exacly the same as it would, when you integrate it into the `nexusUi`.

## Example

```JavaScript
// Test implementation
const Hook = require('nexus-hook').TestHook;
var myHook = new Hook("en");

// run the test hook with the test parameters
myHook.chat("weahter_intent", "whats the weather like in Bremen", (resp) => {
    console.log(resp.answer);
});
```

## Example Output
```
=== Testing Hook ===
Weather hook v0.1.0 loaded
Sorry, I can't find anything.
The weather in Köln is Clouds with $20.6 degrees
The weather in Bremen is Clouds with $18.43 degrees
The weather in Hamburg is Clouds with $18.6 degrees
Weather hook v0.1.0 loaded
Ich habe keine informationen zu deiner Suche gefunden
Das Wetter in Bremen ist Clouds mit $18.43 grad
Das Wetter in Hamburg ist Clouds mit $18.61 grad
Das Wetter in Köln ist Clouds mit $20.6 grad
```

> btNexus is a product of Blackout Technologies, all rights reserved (https://blackout.ai/)
