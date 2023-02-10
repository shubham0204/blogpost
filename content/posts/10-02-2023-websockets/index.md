---
title: "Using Websockets To Run Terminal Commands From Android Apps"
description: "Building Websockets with FastAPI and using them in Android apps"
date: 2023-02-10T08:42:59+05:30
showToc: true
draft: false
---

Websockets are a great tool to establish two-way communication over two devices on a network, and some of the best video calling apps use websockets to transfer audio and video. In this tutorial, we'll build an app that would emulate the computer's terminal. Basically, you will enter commands into the app, they'll get executed on the computer, and the output will be streamed back to the mobile app. Here's a list of frameworks that we'll use to build the app

> Android, Python, FastAPI, Jetpack Compose, Ktor, Kotlin Coroutines

## 1. What are Websockets?
Websocket is a protocol i.e. a set of rules that govern the transfer of data from one device to another within a network of computers. It is similar to the HTTP (HyperText Transfer Protocol) which governs the transfer of hypertexts within a computer network. A hypertext is a text document that contains references to other documents present on the network. HTTP is also used to send binary data, like images, PDFs and other files.

![Client-server architecture with HTTP requests](images/image_1.png#center)

> Client-server architecture with HTTP requests. Source: [ww.oreilly.com](https://www.oreilly.com/library/view/hands-on-network-programming/9781789349863/c099d1ac-9bb2-46cb-9222-339617015438.xhtml)

HTTP assumes a client-server architecture. A client is any device that receives/sends data to the server. While reading this post, the LinkedIn app requested the data of this post (the written text, banner image, details of writer) from a server that has access to the details of all users, stored in a central database. The client, usually a mobile app, website or a script, sends a 'request' to the server and in-turn the server returns a 'response' to the client which contains the information the client requested. In order to get some data (ex. fetch details of a user) from the server, the client initiates a GET request, a type of request in the HTTP, whereas if the client wishes to send some data to the server (for ex. to register a new user), a POST request is initiated.

So, could we use HTTP to design a one-to-one video calling app? Probably not, as the frames generated at 30 FPS are too fast for the HTTP to transfer data between the two devices. There is some overhead when a GET/POST request is sent, both on the client and server sides. For video calling, or messaging apps, we need to establish a realtime connection between both the devices which is fast and has less overhead.

![A websocket connection.](images/image_2.png#center)

> A websocket connection. Source: [Wallarm: A Simple Explanation of What a Websocket is](https://www.wallarm.com/what/a-simple-explanation-of-what-a-websocket-is)

Websockets provide full-duplex communication i.e. two-way over a single connection. The communication between client and the server is also two-way, but it creates different connections for each interaction. Websockets are supported by HTTP, and we can upgrade a HTTP connection to the websocket connection by sending additional details (called headers). You can find dozens of tutorials on the internet that use websockets to create real-time applications like chat/messaging, video calling and establishing wireless connections to media devices.


## 2. Creating a Server To Execute Commands

We need a server with whom our app can establish a websocket connection. The server would run the command received from the client (app) and return the output of the execution back to the client as a response. We'll be using FastAPI to create a websocket and to host it temporarily on the computer.

```python
from fastapi import FastAPI
from fastapi import WebSocket
from command_processor import run
import os

# Create an instance of FastAPI
app = FastAPI( title="Command Processor" )

@app.get( "/pwd" )
async def get_working_dir():
    """
    Get the current working directory in which the server is running
    :return: Path to the current working directory
    """
    return os.getcwd()

@app.websocket( "/run" )
async def command_run_websocket( socket : WebSocket ):
    # Block the execution until socket.accept() completes its task
    await socket.accept()
    # Initiate an infinite loop to listen and send messages across
    # the socket
    while True:
        # Wait for socket.receive_text() to complete
        command = await socket.receive_text()
        # Run the command
        for line in run( command ):
            # Send each line of the output to the client
            await socket.send_text( line )
```

* The `get_working_dir` method is used to get the current working directory in which the server resides. The app will make a request to this method on its start, to make a 'terminal'-like appearance like `$ <current_dir> : <command>` that we have in Windows CMD and Bash.
* `async def` describes methods/functions that do not block the main thread during their execution. Their execution is simply shifted on some other worker thread. The `await` keyword is used with a Coroutine object and it waits for the coroutine to execute and return a result.
* In the above snippet, the `await` keyword is used with `socket.receive_text()` , `socket.accept()` and `socket.send_text()` methods whose outputs are necessary to maintain the flow of the program's execution. They are not fire-and-forget type of methods.

The next step would be host our websocket on a local server using uvicorn , a ASGI web server implementation. ASGI (Asynchronous Server Gateway Interface) servers are different from WSGI (Web Server Gateway Interface) as they expose async functions to the server that are non-blocking in nature.

To launch the server, we run the following command in the project directory,

```
$ uvicorn main:app --host <host_ip> --port 8000
```

where `host_ip` is the IP address of your computer in the network to which it is connected. Make sure that the mobile app and the computer are connected over the same network so that they can discover each with their private IPs. You may use tools like `ipconfig` on Windows and `ifconfig` on Unix-systems to get the IP address. Once executed, you should get the following output,

```
INFO:     Started server process [6432]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://<host_ip>:8000 (Press CTRL+C to quit)
```

Keep the server running, and we'll now move on to the Android app.

## 3. Building the Android App
The Android app will perform the following operations in sequence to emulate the computer's terminal:

1. Send a GET request to the server's `get_working_dir` method to fetch the path of the working directory. We'll store this result and suffix it to each output of the subsequent commands provided by the user.
2. Connect with the websocket which is defined by the method `command_run_websocket` using Ktor's `client.ws` command. We also need to use 'Upgrade Headers' to initiate a websocket connection.
3. Now when the user enters a command, we send to the server through the websocket and the output of the command's execution is returned.

Before jumping into the code, verify that the following Maven dependencies are added in your app's `build.gradle` ,

```
implementation 'io.ktor:ktor-client-android:1.5.0'
implementation 'io.ktor:ktor-client-websockets:1.5.0'
implementation 'io.ktor:ktor-client-okhttp:1.5.0'
```

Next, we create a new class `CommandProcessor.kt` that uses Ktor to connect with the websocket and contains other helper methods,

```kotlin
class CommandProcessor(
    scheme: String,
    host: String,
    port: String,
    private val commandsFlow : MutableSharedFlow<String>,
    private val outputFlow : MutableSharedFlow<String>
) {

    // scheme: http or https
    // host: IP address of the system where the websocket is hosted
    // port: The port over which the websocket is served
    private val websocketUrl = Url( "$scheme://$host:$port/run" )
    private val pwdUrl = Url( "$scheme://$host:$port/pwd" )

    private val ktorClient = HttpClient( OkHttp ){ install( WebSockets ) }
    private val ioScope = CoroutineScope( Dispatchers.IO )

    fun connect() {
        ioScope.launch {
            setupWebSocket()
        }
    }

    fun getWorkingDirectory( callback : (String) -> Unit ) {
        ioScope.launch{
            callback( getCurrentWorkingDir() )
        }
    }

    private suspend fun getCurrentWorkingDir() : String {
        return ktorClient.get( pwdUrl )
    }

    private suspend fun setupWebSocket() {
        ktorClient.ws( HttpMethod.Get ,
            websocketUrl.host ,
            websocketUrl.port ,
            websocketUrl.encodedPath
        ) {
            awaitAll(
                // Non blocking function for sending data/commands
                async {
                    commandsFlow.collect{
                        outgoing.send( Frame.Text( it ) )
                    }
                } ,
                // Non blocking function to receive data/commands
                async {
                    incoming.consumeEach {
                        if( it is Frame.Text ) {
                            outputFlow.emit( it.readText().removeSuffix( "\n" ) )
                        }
                    }
                }
            )
        }
    }

}
```

* The `CommandProcessor` class takes in the details of the websocket connection like the `schema`, `host` and the `port`. Moreover, it also takes two `MutableSharedFlow` that are a part of Kotlin's `Flow` API which enables asynchronous streaming of data. Think of them as two channels, where `commandsFlow` takes commands (as strings) from the UI and delivers them to the websocket connection and `outputFlow` receives the output from the websocket and takes it to the UI.
  
* Next, we create a `ktorClient` and install `WebSockets` as a `Feature`. We'll also need a `CoroutineScope` under which the non-blocking functions would execute. Ktor is built around Coroutines and can handle asynchronous operations efficiently.
  
* `awaitAll` here is similar to Python's `await`, the difference being that `awaitAll` can take multiple jobs for waiting and blocks execution until all given jobs have been completed.

For the rest of the code, we can use Jetpack Compose to build the UI quickly and add some dark colors to get the terminal feel.

![The app's look](images/image_3.jpeg#center)

We split the UI of the app into two major `Composable`, one which takes the input command with an 'Execute' button and another, which shows the output. The `CommandInput` composable contains `TextField` and a `Button`,

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CommandPromptInput() {
    var commandText by rememberSaveable{ mutableStateOf( "" ) }
    Row( modifier = Modifier
        .height(60.dp)
        .fillMaxWidth() ) {
        TextField(
            value = commandText,
            onValueChange = { commandText = it } ,
            modifier = Modifier.fillMaxWidth().weight(1.0f).height(60.dp),
            placeholder = { Text(text = "Enter command here..." , fontFamily = codeFont ) } ,
            shape = RectangleShape ,
            colors = TextFieldDefaults.textFieldColors(
                containerColor = Color.White ,
                textColor = Color.Black ,
                disabledIndicatorColor = Color.Transparent ,
                focusedIndicatorColor = Color.Transparent ,
                unfocusedIndicatorColor = Color.Transparent
            ) ,
            textStyle = TextStyle( fontFamily = codeFont )
        )
        Button(
            onClick = {
                CoroutineScope( Dispatchers.Default ).launch {
                    Log.e( "APP" , "Command emitted: $commandText")
                    commandsFlow.emit( commandText )
                    outputFlow.emit("\n $currentWorkingDirectory > $commandText")
                    commandText = ""
                }
            } ,
            shape = RectangleShape ,
            colors = ButtonDefaults.buttonColors( containerColor = Color.White ) ,
            modifier = Modifier.height(70.dp).width(70.dp)
        ) {
            Icon( Icons.Default.PlayArrow, "Execute" , tint = Color.Black )
        }
    }
}
```

The `CommandOutput` composable receives a `LinkedList<String>` which holds the output of each command as a line. Instead of maintaining a single `Text` and concatenating new output lines to the existing text, we create a `LazyColumn` which holds outputs as items of a list. This technique does not put the burden on memory if a large number of commands are executed.

```kotlin
@Composable
fun CommandOutput( modifier: Modifier ) {
    val outputLines by displayTextLiveData.observeAsState()
    CommandOutputList( commandOutputLines = outputLines!! ,  modifier = modifier )
}

@Composable
fun CommandOutputList( commandOutputLines : LinkedList<String> , modifier: Modifier ) {
    val listState = rememberLazyListState()
    LaunchedEffect( commandOutputLines.size ) {
        if( commandOutputLines.size != 0 ) {
            listState.animateScrollToItem( commandOutputLines.size - 1 )
        }
    }
    LazyColumn( modifier = modifier , state = listState ) {
        items( commandOutputLines ) {
            Text(text = it ,
                modifier = Modifier.fillMaxWidth() ,
                color = Color.White ,
                fontSize = 8.sp ,
                fontFamily = codeFont ,
            )
        }
    }
}
```

After building `Composable`s for the UI and the `CommandProcessor`, we can assemble all components in the `onCreate` method of `MainActivity`,

```kotlin
class MainActivity : ComponentActivity() {

    private val commandsFlow : MutableSharedFlow<String> = MutableSharedFlow()
    private val outputFlow : MutableSharedFlow<String> = MutableSharedFlow()

    private val scheme = "http"
    private val host = "192.168.220.103"
    private val port = "8000"

    private val webSocketHandler = CommandProcessor( scheme , host , port , commandsFlow , outputFlow )
    private var currentWorkingDirectory = host
    private val displayText = LinkedList<String>()
    private val displayTextLiveData = MutableLiveData<LinkedList<String>>( LinkedList())

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            CommandProcessorTheme {
                CommandPromptUI()
            }
        }
        webSocketHandler.connect()
        webSocketHandler.getWorkingDirectory{
            currentWorkingDirectory = it
        }
        CoroutineScope( Dispatchers.Main ).launch {
            outputFlow.collect {
                displayText.add( it )
                val newList = LinkedList( displayText )
                displayTextLiveData.value = newList
            }
        }
    }
```

`outputFlow.collect` will be called whenever a new value is emitted from `CommandProcessor`. We add it to a new `LinkedList` and update the value of `displayTextLiveData` so the coupled UI element in `CommandOutput` can be updated. Build the app and test it on a physical device which is connected to the same network on which the server-running computer is visible.

<html>
<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/cP3kOtwUzfA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</center>
</html>

* **Limitations**: The terminal sessions are not persistent as each command runs individually in a new subprocess. Also, the sessions get destroyed when the app is closed. Also, the local server has to stay activate to keep the websocket connection alive.

* **Uses**: While working on a project, if you need to run some commands repeatedly, you can run them in your mobile terminal without creating a new desktop terminal window.
  
## Thanks

Websockets are a great tool for app developers as they can enable realtime communication abilities. Combine them with the ease of FastAPI and Python, and you're ready to get the best of both worlds! Thanks for reading and have a nice day ahead!