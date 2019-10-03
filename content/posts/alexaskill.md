+++
draft = false
date = 2019-10-03T07:36:56-07:00
title = "Alexa : Build me a skill"
slug = ""
+++

The other day I was asked to look in to how we could build an Alexa skill that interacted with an onsite service. This was a lot more fun than expected and I wanted to share it as a blog post.

## Creating A Skill
First things first, head over to the [Alexa developer](https://developer.amazon.com/en-US/alexa) page and sign up for an account.

Once in the console, we will create a new skill. Let's go with an audio only simple python script hosted in AWS for us. To do so, select a 'Alexa-Host (Python)' skill. This is going to give you an all in one environment including a code editor and quick deployment of your python script as a Lambda function. As it is a Lambda, the code will run without the need for a server or any other concerns of yours.

This is where things start to get a bit more complicated and seems overwhelming at first but really its not too bad. There are three main that we will focus on for the demo: Build, Code, and Test.

## Build - The Voice Model
In an Alexa skill, you are taking verbal commands and mapping them to your functions. The build screen is focused on building out the model to do so. Let's say you wanted a command like: "tell mr robot to save the world". In this command "Mr Robot" would be the invocation and "Save the world" would be the intent utterance.

To set an invocation, click the invocation on the menu then simply give it a name. The page has some details but the key is to set a two word command.

The intents are going to be all the actions we want to map. For each intent, you can have one or more utterances. If my command was the SaveTheWorldIntent then I might have an utterance of "save the world" but maybe also "save world", "my world needs saving", and "save all the things". This can be done with the following:

1. Click the "Add" button next to Intents
2. Enter a name: SaveTheWorldIntent
3. Then enter each of the utterances (Ignore slots for now, I'll come back to those)
4. Now just save and build your model from the menu at the top.

## Code - Python Lambda
There is definitely a robust Alexa API and Python can do far more than this but I don't want Alexa to perform any logic for me. I just want it to be a proxy and call an API which makes my code quite simple. If you want to do more, feel free to run with it.

Since we are calling an API, let's add the [requests](https://pypi.org/project/requests/) library to our project. We then add a procedure for the API and finally our handler logic:

1. Open requrements.txt
2. Add a line:
{{< highlight text >}}
requests==2.22.0
{{< /highlight >}}
3. Open lambda_function.py
4. Add the import statement
{{< highlight python >}}
import requests
{{< /highlight >}}
5. Add some global configuration variables
{{< highlight python >}}
api_url=http://mypublicendpoint/api
api_username=MyUser
api_password=MySecurePassword
{{< /highlight >}}
6. Add a callAPI function
{{< highlight python >}}
def callApi(api):
    resp = requests.get(api_url+api,auth=(api_username,api_password))
    if resp.status_code != 200:
        # This means something went wrong.
        speak_output="I failed you!"
    speak_output=resp.text
    return speak_output
{{< /highlight >}}
7. Add the snippet:
{{< highlight python >}}
class SaveTheWorldIntentHandler(AbstractRequestHandler):
    """Handler for Save The World."""
    def can_handle(self, handler_input):
        # type: (HandlerInput) -> bool
        return ask_utils.is_intent_name("SaveTheWorldIntent")(handler_input)

    def handle(self, handler_input):
        # type: (HandlerInput) -> Response

        speak_output = callApi('/savetheworld/?userId={}'.format(handler_input.request_envelope.session.user.user_id))

        return (
            handler_input.response_builder
                .speak(speak_output)
                .response
        )
{{< /highlight >}}
8. Add the handler to the handler list at the end
{{< highlight python >}}
sb.add_request_handler(SaveTheWorldIntentHandler())
{{< /highlight >}}

The snippet above has a few important items to look at. First, anywhere that references SaveTheWorld must map directly (including case) to the name of the intent. Inside are two functions, can_handle to return if it handles the provided intent and then handle to perform whatever is needed. In this case, that action is to call an API then speak whatever the api says. I also include the user_id in the API call for reference to the unique id for the given user.

## Test
If all went according to plan, you should now have an extremely simple skill that will run or almost. Pop over to the Test tab and enable testing in Development. Now there is an Alexa Simulator that lets you work right there in the console with your new skill.

In the box either hold the mic and speak your commands or just type : tell mr robot to save the world. If everything went perfectly you'll get the proper response. More likely, something went wrong and you got one of the following:

- You just triggered SaveTheWorldIntent -> This means you likely have a typo in your handler or missed adding the handler at the end.
- \<Audio only response\> / Error chime -> This means something failed on the Lambda side. If that happens, you can pop back to the Code tab then in the bottom left click "Logs: Amazon CloudWatch" which open ups the CloudWatch log viewing tool. Here just click "Search Log Group" and scroll through to find your error.
- JSON or other API level error message -> You likely successfully hit your API but failed to send the right inputs.
- Other unknown or wrong response -> Check your model utterances.

You can also log in to your dev account with your Alexa device (or even the phone app) and the skill is available for testing.

## Advanced - Slots and Collecting Data
On the Build page, there was a concept of slots. A slot can serve two main purposes for you. One is to allow multiple values in a one utterance (I.E. I want a large puppy and  I want a small puppy can become I want a {size} puppy). The second is to be more flexible and collect some data.

Let's say you want have your widget learn your name. This is easy enough:

1. Add a new intent called SayMyNameIntent
2. Add an utterance of "my name is {name}"
3. Click to Create a new slot when prompted
4. Type "name" for the slot name and click the +
5. For the slot type, choose AMAZON.FirstName
6. Save and Build your model
7. Go to your lambda_function.py in the Code tab
8. Add our new intent handler. Note the new line of grabbing the variable via "ask_utils.get_slot_value"
{{< highlight python >}}
class SayMyNameIntentHandler(AbstractRequestHandler):
    """Handler for Save The World."""
    def can_handle(self, handler_input):
        # type: (HandlerInput) -> bool
        return ask_utils.is_intent_name("SayMyName")(handler_input)

    def handle(self, handler_input):
        # type: (HandlerInput) -> Response

        name = username = ask_utils.get_slot_value(handler_input,"name")

        speak_output = "Your name is "+name

        return (
            handler_input.response_builder
                .speak(speak_output)
                .response
        )
{{< /highlight >}}
9. Add the handler to the handler list at the end
{{< highlight python >}}
sb.add_request_handler(SayMyNameIntentHandler())
{{< /highlight >}}
10. Save and Deploy

## Have Fun!
This is just a start but is enough to be dangerous. Be careful and secure it properly.
