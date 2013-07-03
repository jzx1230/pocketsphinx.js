PocketSphinx.js
---------------
### Speech Recognition in JavaScript

PocketSphinx.js is a speech recognizer that runs entirely in the web browser. It is built on:

* a speech recognizer written in C ([PocketSphinx](http://cmusphinx.sourceforge.net/)) converted into JavaScript using [Emscripten](https://github.com/kripken/emscripten),
* an audio recorder using the Web Audio API.

You can try it on the project page: <http://syl22-00.github.io/pocketsphinx.js> and have a look at our [FAQ](https://github.com/syl22-00/pocketsphinx.js/wiki/FAQ).

Table of contents:

1. Overview
2. Compilation of `pocketsphinx.js`
3. API of `pocketsphinx.js`
4. Using `pocketsphinx.js` inside a Web Worker with `recognizer.js`
5. Wiring `recognizer.js` to the audio recorder
6. Live demo
7. Test suite
8. Notes about speech recognition and performance
9. License

# 1. Overview

This project includes several components that can be used independently:

* `pocketsphinx.js`, a JavaScript library generated by emscripten which is basically PocketSphinx wrapped to provide a simpler API, and compiled into JavaScript.
* `recognizer.js`, a wrapper around `pocketsphinx.js` inside a Web Worker to unload the UI thread from downloading and running the large JavaScript file and running the costly speech recognition process.
* `audioRecorder.js`, an audio recording library, based on [Recorderjs](https://github.com/mattdiamond/Recorderjs). It converts the recorded samples to the proper sample rate and passes them to the recognizer.
* `callbackManager.js`, a small utility to interact with Web Workers with calls and callbacks rather than message passing.

The file `webapp/live.html` illustrates how these work together in a real application, that is a good starting point. Make sure you load it through a web server or start Chrome with `--disable-web-security`.

# 2. Compilation of `pocketsphinx.js`

A prebuilt version of `pocketsphinx.js` is available in `webapp/js`, or you can build it yourself. Below is the procedure on Linux (and probably Mac OS X). On Windows, refer to the emscripten manual.

## 2.a Compilation with the default acoustic model

You will need:

* [emscripten](https://github.com/kripken/emscripten) (which implies also node.js and LLVM), 
* [CMake](http://www.cmake.org/).

The build is a classic CMake cross-compilation, using the toolchain provided by emscripten:

    $ cd .../pocketsphinx.js # This folder
    $ mkdir build
    $ cd build
    $ cmake -DEMSCRIPTEN=1 -DCMAKE_TOOLCHAIN_FILE=path_to_emscripten/cmake/Platform/Emscripten_unix.cmake ..
    $ make

This generates `pocketsphinx.js`. At this point, optimization level is hard-coded, so modify `CMakeLists.txt` directly if you would like to change it.

## 2.b Compilation with custom acoustic models

The compilation process packages the acoustic models inside the resulting JavaScript file. If you would like to use your own models, you should specify where they are when running `cmake`. For that, place all models you want to package inside a `HMM_BASE` folder. Each model being in its own folder inside `HMM_BASE`:

    $ cmake -DEMSCRIPTEN=1 -DCMAKE_TOOLCHAIN_FILE=path_to_emscripten/cmake/Platform/Emscripten_unix.cmake -DHMM_BASE=/path/to/models -DHMM_FOLDERS="model1;model2;..." ..

If you only need to package one model, you can also do:

    $ cmake -DEMSCRIPTEN=1 -DCMAKE_TOOLCHAIN_FILE=path_to_emscripten/cmake/Platform/Emscripten_unix.cmake -DHMM_BASE=/path/to/models -DHMM_FOLDERS=model ..

Make sure the files of the acoustic model are directly inside the `HMM_FOLDERS`:

    $ cd /path/to/models
    $ ls *
    model1:
    feat.params  mdef  means  sendump  transition_matrices  variances
    
    model2:
    feat.params  mdef  means  sendump  transition_matrices  variances

Please note:

* If you want to package your own models, you need to set both `HMM_BASE` and `HMM_FOLDERS`.
* By default, the first provided acoustic model will be loaded if none is specified before the recognizer is initialized. The model can be selected by giving the `"-hmm"` parameter. See upcoming sections for how to specify recognizer parameters.
* Make sure you optimize the size of your models (`mdef` in binary format, `sendump` instead of `mixture_weights`, see PocketSphinx docs).


# 3. API of `pocketsphinx.js`

You can interact with `pocketsphinx.js` directly if you need to, but it is probably easier to build your application against the API of `recognizer.js` described in a later section.

## 3.1 Principles

The file `pocketsphinx.js` can be directly included into an html file but as it is fairly large (a few MB, depending on the optimization level used during compilation), downloading and loading it will take time and affect the UI thread. So, as explained later, you should use it inside a Web Worker, for instance using `recognizer.js`.

You should probably have a look at emscripten's docs to understand how to interact with emscripten-generated JavaScript. For instance, to retrieve the recognizer's state:

    $ var getState = Module.cwrap('psGetState');
    $ var state = getState();

Calls to `pocketsphinx.js` functions are synchronous, that's also why you probably need to load it in a Web Worker, as explained in later sections. The recognizer can be at any of these states:

* UNINITIALIZED = 0, when the file is just loaded.
* INITIALIZED = 1, after a successful call to the initialization function, and any time it is ready to perform actions.
* ENTERING_GRAMMAR = 3, when it is getting grammar data.
* RECORDING = 4, when it is getting audio data.

Most calls return an error code, which can be one of the following:

* SUCCESS = 0, if the action was performed successfully.
* BAD_STATE = 1, if the current state does not allow the action.
* BAD_ARGUMENT = 2, if the argument provided is invalid.
* RUNTIME_ERROR = 3, if there is a runtime error in the recognizer.


## 3.2 Calls

In order to allocate and free memory, we import `malloc` and `free` first:

    var c_malloc = Module.cwrap('malloc', 'number', ['number']);
    var c_free = Module.cwrap('free', 'number', ['number']);

### a. psGetState

Retrieves the current state:

    var psGetState = Module.cwrap('psGetState');
    var state = psGetState(); // Gets the current state of the recognizer

### b. psGetHyp

Retrieves the current hypothesis (the recognized string up to this point). It can be called at any time. If it is called during recording, this is the hypothesis up to that point, otherwise, it is the hypothesis of the last recorded sentence, after the second pass (more accurate than hypothesis during recognition):

    var psGetHyp = Module.cwrap('psGetHyp', 'string');
    var hyp = psGetHyp(); // Gets the current hypothesis

### c. psSetParam

Sets a PocketSphinx recognizer parameter. Parameters should be given before the recognizer is initialized with `psInitialize`. The parameters will be passed directly to pocketsphinx. For instance:

    var psSetParam = Module.cwrap('psSetParam', 'number', ['number','number']);
    var key = "-fwdflat"; // Must match a valid pocketsphinx command-line parameter
    var value = "no"; // Must be a valid value for the parameter
    // We create C strings from the JavaScript strings:
    var key_ptr = Module.allocate(intArrayFromString(key), 'i8', ALLOC_STACK); 
    var value_ptr = Module.allocate(intArrayFromString(value), 'i8', ALLOC_STACK);
    var output = psSetParam(key_ptr, value_ptr); // Returns 0 if successful, see possible error codes above
    var psInitialize = Module.cwrap('psInitialize');
    output = psInitialize(); // Returns 0 if successful, see possible error codes above

If you have included several acoustic models when compiling `pocketsphinx.js`, you can select which one should be used by setting the `"-hmm"` parameter. Say you have two models, one for English, one for French, and you have compiled the library with `-DHMM_FOLDERS="english;french"`, you can initialize the recognizer with the French model by setting the correct value before the call to `psInitialize`:

    var key = "-hmm";
    var value = "french";
    var key_ptr = Module.allocate(intArrayFromString(key), 'i8', ALLOC_STACK); 
    var value_ptr = Module.allocate(intArrayFromString(value), 'i8', ALLOC_STACK);
    var output = psSetParam(key_ptr, value_ptr);
    var psInitialize = Module.cwrap('psInitialize');
    output = psInitialize(); // Returns 0 if successful, see possible error codes above

If you do not call `psSetParam` with `"-hmm"`, or call it with an incorrect value, the first model in the list will be used (here, `english`).

### d. psInitialize

Initializes the recognizer. This can take a few seconds, that's one of the reasons why you should use `pocketsphinx.js` inside a Web Worker.

    var psInitialize = Module.cwrap('psInitialize');
    var output = psInitialize(); // Returns 0 if successful, see possible error codes above

Note that if the recognizer is already initialized, it will return `BAD_STATE`.

### e. psStartGrammar

At this point, `poketsphinx.js` can only use Finite State Grammars (FSG) as language models. To input grammars, you should be in the `INITIALIZED` state, call `psStartGrammar`, add transitions with `psAddTransition`, then call `psEndGrammar`. Before that, you must have added all words used in the grammar with `psAddWord`, described below.

    var psStartGrammar = Module.cwrap('psStartGrammar', 'number', ['number']);
    var numStates = 1; // Number of states in the FSG
    var output = psStartGrammar(numStates); // Returns 0 if successful, see possible error codes above

### f. psEndGrammar

Once all transitions have been added, this should be called to close the grammar and retrieve its id. The given id will be used to switch to that grammar. A call to `psEndGrammar`, if successful, always switch the recognizer to the grammar so that no call to `psSwitchGrammar` is necessary to use it right away. It takes as argument a pointer to an allocated 4 byte-integer that will be set to the value of the id of the newly added grammar:

    var psEndGrammar = Module.cwrap('psEndGrammar', 'number', ['number']);
    var id_ptr = c_malloc(4); // Creates a pointer to receive the value of the id
    var out = psEndGrammar(id_ptr); // Returns 0 if successful, see possible error codes above
    var id = getValue(id_ptr, 'i32'); // Gets the value of the id
    c_free(id_ptr); // Frees the memory

### g. psSwitchGrammar

The recognizer can switch, at runtime, between grammars, using their id provided by `psEndGrammar`. For instance:

    var psSwitchGrammar = Module.cwrap('psSwitchGrammar', 'number', ['number']);
    var myGrammarId = 5; // Value that was given by call to psEndGrammar
    var out = psSwitchGrammar(myGrammarId); // Returns 0 if successful, see possible error codes above

### h. psAddWord

All words used in grammars must be present in the pronunciation dictionary. Refer to the [CMU Pronunciation Dictionary site](http://www.speech.cs.cmu.edu/cgi-bin/cmudict) if you are not familiar with it.

    var psAddWord = Module.cwrap('psAddWord', 'number', ['number','number']);
    var word = "HELLO";
    var pronunciation = "HH EH L OW"; // CMU dict syntax
    // We create C strings from the JavaScript strings:
    var word_ptr = Module.allocate(intArrayFromString(word), 'i8', ALLOC_STACK); 
    var pron_ptr = Module.allocate(intArrayFromString(pronunciation), 'i8', ALLOC_STACK);
    var out = psAddWord(word_ptr, pron_ptr); // Returns 0 if successful, see possible error codes above

### i. psAddTransition

Transitions should be added to grammars between the calls to `psStartGrammar` and `psEndGrammar`. For now, we assume states start at `0` and `numStates - 1` is the last state, if the grammar has `numStates` states.

    var psAddTransition = Module.cwrap('psAddTransition', 'number', ['number','number','number']);
    var word = "HELLO"; // The word should already have been added with psAddWord
    var word_ptr = Module.allocate(intArrayFromString(word), 'i8', ALLOC_STACK);
    var from_state = 0; // First state of the transition
    var to_state = 1; // Second state of the transition
    var out = psAddTransition(from_state, to_state, word_ptr); // Returns 0 if successful, see possible error codes above

### j. psStart

Starts the recognition process, using the last added grammar or the one set with the last call to `psSwitchGrammar`. The recognizer should be at the `INITIALIZED` state to be able to start recognition.

    var psStart = Module.cwrap('psStart');
    var out = psStart();  // Returns 0 if successful, see possible error codes above

### k. psStop

Stops the recognition process. As the recognizer will run a second pass when this is called, the hypothesis returned by `psGetHyp` is more accurate after than before the call to `psStop`.

    var psStop = Module.cwrap('psStop');
    var out = psStop();  // Returns 0 if successful, see possible error codes above

### l. psProcess

Audio data should be sent to the recognizer using repeated calls to this function, between calls to `psStart` and `psStop`. Data should be 2-byte integers (audio should be 16 kHz, 16 bit for the provided acoustic model. If you choose to use a different acoustic model, you'll need to adjust those accordingly.).

    var psProcess = Module.cwrap('psProcess', 'number', ['number','number']);
    var array = ... // array contains the audio samples
    var buffer = c_malloc(2 * array.length);
    // The buffer must be filled in with calls to setValue
    for (var i = 0 ; i < array.length ; i++)
        setValue(buffer + i*2, array[i], 'i16');
    var out = psProcess(buffer, array.length); // The data and the length of the data must be given
    c_free(buffer);

# 4. Using `pocketsphinx.js` inside a Web Worker with `recognizer.js`

Using `recognizer.js`, `pocketsphinx.js` is downloaded and executed inside a Web Worker. The file is located in `webapp/js/`, both `recognizer.js` and `pocketsphinx.js` must be in the same folder at runtime. It is intended to be loaded as a new Web Worker:

    var recognizer = new Worker("js/recognizer.js");

You can then interact with it using messages.

## 4.1 Incoming Messages

Messages posted to the recognizer worker might include the following attributes:

* `command`, command to be executed,
* `data`, data to be passed to the command,
* `callbackId`, id to be passed to the outgoing message, might be used to trigger a callback.

## 4.2 Outgoing Messages

The worker sends messages back to the UI thread, either to call back when actions have been performed, report errors or send periodic information such as the current recognition hypothesis.

Messages posted by the recognizer worker might include:

* `status`, which can be either `done` or `error`,
* `command`, the command that sent the message,
* `code`, an error code,
* `id`, a callback id that was given in the received incoming message,
* `data`, additional data that the callback function might make use of,
* `hyp`, the current recognition hypothesis,
* `final`, a boolean that indicates whether the hypothesis is final (sent after call to `stop`).

## 4.3 API description

### a. Error codes

The error codes returned in messages posted back from the worker can be:

* the error code returned by `pocketsphinx.js` as explained previously,
* or one of the following strings:
    * "js-data", if the provided data are invalid,
    * "js-no-recognizer", if the recognizer is not initialized.

### b. Initialization

Once the worker is created, the recognizer must be initialized:

    var id = 0; // This value will be given in the message received after the action completes
    recognizer.postMessage({command: 'initialize', callbackId: id});

Once it is done, the recognizer will post a message back, for instance:

* `{status: "done", command: "initialize", id: clbId}`, if successful, where `clbId` is the callback id given in the original command message.
* `{status: "error", command: "initialize", code: initStatus}`, if there is an error, where `initStatus` is the value returned by the call to `psInitialize`, see above for possible values.

Recognizer parameters to be passed to `PocketSphinx` can be given in the call to `initialize`. For instance:

    recognizer.postMessage({command: 'initialize', callbackId: id, data: [["-hmm", "french"], ["-fwdflat", "no"]]});

This will set the `pocketsphinx` command-line parameter `"-fwdflat"` to `no` and initialize the recognizer with the acoustic model `french`, assuming `pocketsphinx.js` was compiled with that model.

### c. Adding words

Words to be recognized must be added to the recognizer before they can be used in grammars. See previous sections to know more about the format of dictionary items. Words can be added at any time after the recognizer is initialized, and several words can be added at once:

    var words = [["ONE", "W AH N"], ["TWO", "T UW"], ["THREE", "TH R IY"]]; // An array of pairs [word, pronunciation]
    recognizer.postMessage({command: 'addWords', data: words, callbackId: id});

The message back could be:

* `{id: clbId}`, the provided callback id, if given, as explained before, if successful.
* `{status: "error", command: "addWords", code: code}`, if error, where possible values of the error code was described above.

### d. Adding grammars

As described previously, any number of grammars can be added. The recognizer can then switch between them. A grammar can be added at once using a JavaScript object that contains the number of states and an array of transitions, for instance:

    var grammar = {numStates: 3, transitions: [{from: 0, to: 1, word: "HELLO"}, {from: 1, to: 2, word: "WORLD"}]};
    recognizer.postMessage({command: 'addGrammar', data: grammar, callbackId: id});

All words must have been added previously using the `addWords` command.

In the message back, the grammar id assigned to the grammar is given. It can be used to switch to that grammar. So the message, if successful, would be like `{id: clbId, data: id, status: "done", command: "addGrammar"}`, where `id` is the id of the newly created grammar. In case of errors, the message would be as described previously.

### e. Starting recognition

The message to start recognition should include the id of the grammar to be used:

    recognizer.postMessage({command: 'start', data: id}); // where id is the id of a previously added grammar.

### f. Processing data

Audio samples should be sent to the recognizer using the `process` command:

    recognizer.postMessage({command: 'process', data: array}); // array is an array of audio samples

Audio samples should be 2-byte integers, at 16 kHz.

While data are processed, hypothesis will be sent back in a message in the form `{hyp: "RECOGNIZED STRING"}`.

### g. Ending recognition

Recognition can be simply stopped using the `stop` command: 

    recognizer.postMessage({command: 'stop'});

It will then send a last message with the hypothesis, marked as final (which means that it is more accurate as it comes after a second pass that was trigered by the `stop` command). It would look like: `{hyp: "FINAL RECOGNIZED STRING", final: true}`.

## 4.4 Using `CallbackManager`

In order to facilitate the interaction with the recognizer worker, we have made a simple utility that helps associate callbacks to be executed when the worker posts a message responding to a command you sent. You can find `callbackManager.js` in `webapp/js`.

To use it, first create a new instance of CallbackManager:

    var callbackManager = new CallbackManager();

When you post a message to the recognizer worker and want to associate a callback function to it, you first add your callback function to the manager, which gives you a callback id in return:

    recognizer.postMessage({command: 'addWords', data: words, callbackId: callbackManager.add(function() {alert("Words added");})});

In the `onmessage` function of your worker, use the callback manager to check and trigger callback functions:

    recognizer.onmessage = function(e) {
        if (e.data.hasOwnProperty('id')) {
            // If the message has an id field, it
            // means that we might have a callback associated
            var clb = callbackManager.get(e.data['id']);
            var data = {};
            // As mentinned previously, additional data can be passed to the callback
            // such as the id of a newly added grammar
            if(e.data.hasOwnProperty('data')) data = e.data.data;
            if(clb) clb(data);
        }
        // Check for other message types here
    };

Check `live.html` in `webapp` for more examples.


## 4.5 Detecting when the Worker is ready

When a new worker is instanciated, it immediately returns a worker object, but the actual download of the JavaScript files might take some time, especially in our case where `pocketsphinx.js` is fairly large. One way of detecting whether the files are fully downloaded and loaded is to post a first message right after it is instanciated and wait for a message back from the worker.

    var recognizer;
    function spawnWorker(workerurl, onReady) {
        recognizer = new Worker(workerurl);
        recognizer.onmessage = function(event) {
            // onReady will be called when there is a message
            // back
            onReady(recognizer);
        };
        recognizer.postMessage({});
    };

Then, after the first message back was received, propers listening to onmessage can be added:

    spawnWorker("js/recognizer.js", function(worker) {
        worker.onmessage = function(e) {
        // Add what you want to do with messages back from the worker
        };
        // Here is a good place to send the 'initialize' command to the recognizer
     });

Of course, the worker must be able to respond to the first message, as we did in `recognizer.js`:

    function startup(onMessage) {
        self.onmessage = function(event) {
            self.onmessage = onMessage;
            self.postMessage({});
        }
    };
    // This function is called first, it triggers
    // a first postmessage, then adds the proper respond to
    // commands: 
    startup(function(event) {
        switch(event.data.command){
            //We deal with commands properly
        }
    });

All these are illustrated in `webapp/live.html` and `recognizer.js`.

# 5. Wiring `recognizer.js` to the audio recorder

We include an audio recording library based on the Web Audio API that accesses the microphone, gets audio samples, converts them to the proper sample rate (16kHz), and sends them to the recognizer. This library is derived from [Recorderjs](https://github.com/mattdiamond/Recorderjs).

Include `audioRecorder.js` in the HTML file and make sure `audioRecorderWorker.js` is in the same folder. To use it, create a new instance of `AudioRecorder` giving it as argument a `MediaStreamSource`. As of Today, only Google Chrome implements it. You also need to set the recognizer attribute to a Recognizer worker, as described above.


    var audio_context = new AudioContext;
    // Callback once user authorises access to the microphone:
    function startUserMedia(stream) {
        var input = audio_context.createMediaStreamSource(stream);
        input.connect(audio_context.destination);
        recorder = new AudioRecorder(input);
        // The recognizer worker must be given to the recorder
        if (recognizer) recorder.recognizer = recognizer;
    };
    navigator.getUserMedia({audio: true}, startUserMedia, function(e) {
        // What happens in case of errors...
    });

Once the recorder is up and running, you can start and stop recording and recognition with:

    // To start recording:
    recorder.start();
    // The hypothesis is periodically sent by the recognizer, as described previously
    // To stop recording:
    recorder.stop();  // The final hypothesis is sent


This is also illustrated in the given live demo, in the `webapp/` folder.

Note that live audio capture is only available on recent versions of Google Chrome, and on many platforms that feature is not usable and only produces silent audio. Hopefully this will be solved soon. You can track progress on the chromium issue tracker:

<https://code.google.com/p/chromium/issues/detail?id=170384> and <http://code.google.com/p/chromium/issues/detail?id=112367>.

# 6. Live demo

The file `webapp/live.html` is an example of live recognition using the web audio API. It works on Chrome, if the Web audio API actually works. Note that we observed the recorded audio to be silent on most (but not all) configuration we have tried.

To build an application, this is a good starting point as it illustrates the different components described in this document. In that demo, three different grammars are available and the app can switch between them.

# 7. Test suite

There is a test suite being developed in `tests/js`, it makes use of [QUnit](http://qunitjs.com). There is a README file inside the folder.

# 8. Notes about speech recognition and performance

If you are not familiar with speech recognition, you might need to take some time to learn some of the concepts, mainly:

* acoustic models (we provide one small model for English but other Sphinx acoustic models can be used as well),
* language models (at this point, we only have the API to input grammars, as FSGs, but API to input statistical language models could be added).

In terms of performance, you should get exaclty the same result as using PocketSphinx compiled on other platforms.

## 8.1 Acoustic model

The `am` folder contains an acoustic model trained with [SphinxTrain](http://cmusphinx.sourceforge.net/wiki/tutorialam). It is built using the [RM1](http://www.speech.cs.cmu.edu/databases/rm1/index.html) corpus, semi-continuous, with 200 senones.

## 8.2 PocketSphinx

PocketSphinx.js ships with PocketSphinx and Sphinxbase version 0.8 with a few modification and fixes. Also, the `model` folder of PocketSphinx (which contains large acoustic and language models) was not included.

# 9. License

PocketSphinx licensing terms are included in the `pocketsphinx` and `sphinxbase` folders. 

The files `webapp/js/audioRecorder.js` and `webapp/js/audioRecorderWorker.js` are based on [Recorder.js](https://github.com/mattdiamond/Recorderjs), which is under the MIT license (Copyright © 2013 Matt Diamond).

The remaining of this software is licensed under the MIT license:

Copyright © 2013 Sylvain Chevalier

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
