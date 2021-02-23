async_flask

Shane Lynn 15/07/2014

Updated to Python 3: 19th-May-2018

===========

Test of asynchronous flask communication with web page. 

This repository is a sample flask application that updates a webpage using a background thread for all users connected.
It is based on the very useful Flask-SocketIO code from Miguel Grinberg.

https://github.com/miguelgrinberg/Flask-SocketIO

To use - please clone the repository and then set up your virtual environment using the requirements.txt file with pip and virtualenv. You can achieve this with:


    git clone https://github.com/shanealynn/async_flask
    cd async_flask
    virtualenv flaskiotest
    ./flaskiotest/Scripts/activate
    pip install -r requirements.txt  #(or in Windows - sometimes python -m pip install -r requirements.txt )



Start the application with:

<code>
python application.py
</code>

And visit http://localhost:5000 to see the updating numbers.


설명 사이트
https://www.shanelynn.ie/asynchronous-updates-to-a-webpage-with-flask-and-socket-io/


Asynchronous updates to a webpage with Flask and Socket.io
27 Comments / blog, python, Software, web / By shanelynn
communications with socket io and flask in python
Update: Updated May 2018 to work with Python 3 on GitHub.

This post is about creating Python Flask web pages that can be asynchronously updated by your Python Flask application at any point without any user interaction. We’ll be using Python Flask, and the Flask-SocketIO plug-in to achieve this. The working application is hosted on GitHub.

What I want to achieve here is a web page that is automatically updated for each user as a result of events that happened in the background on my server system. For example, allowing events like a continually updating message stream, a notification system, or a specific Twitter monitor / display. In this post, I show how to develop a bare-bones Python Flask application that updates connected clients with random numbers. Flask is an extremely lightweight and simple framework for building web applications using Python.

Flask logo
If you haven’t used Flask before, it’s amazingly simple, and to get started serving a very simple webpage only requires a few lines of Python:

# Basic Flask Python Web App
from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello():
    return "Hello World!"
if __name__ == "__main__":
    app.run()
Running this file with python application.py will start a server on your local machine with one page saying “Hello World!” A quick look through the documentation and the first few sections of the brilliant mega-tutorial by Miguel Grinberg will have you creating multi-page python-based web applications in no time. However, most of the tutorials out there focus on the production of non-dynamic pages that load on first accessed and don’t describe further updates.

For the purpose of updating the page once our user has first visited, we will be using Socket.io and the accomanying Flask addon built by the same Miguel Grinberg, Flask-Socketio (Miguel appears to be some sort of Python Flask God). Socket IO is a genius engine that allows real-time bidirectional event-based communication. Gone are the days of static HTML pages that load when you visit; with Socket technology, the server can continuously update your view with new information.


For Socket.io communication, “events” are triggered by either the server or connected clients, and corresponding callback functions are set to execute when these events are detected. Implementing event triggers or binding event callbacks are very simply implemented in Flask (after some initial setup) using:

# Basic blask server to catch events
from flask import Flask, render_template
from flask.ext.socketio import SocketIO, emit
app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret!'
socketio = SocketIO(app)
@socketio.on('my event')                          # Decorator to catch an event called "my event":
def test_message(message):                        # test_message() is the event callback function.
    emit('my response', {'data': 'got it!'})      # Trigger a new event called "my response" 
                                                  # that can be caught by another callback later in the program.
if __name__ == '__main__':
    socketio.run(app)
Four events are allowed in the @socketio.on() decorator – ‘connect’, ‘disconnect’, ‘message’, and ‘json’. Namespaces can also be assigned to keep things neatly separated, and the send() or emit() functions can be  used to send ‘message’ or custom events respectively – see the details on Miguel’s page.

On the client side, a little bit of JavaScript wizardry with jQuery is used to handle incoming and trigger outgoing events. I would really recommend the JavaScript path on CodeSchool if you are not familiar with these technologies.

// Client Side Javascript to receive numbers.
$(document).ready(function(){
    // start up the SocketIO connection to the server - the namespace 'test' is also included here if necessary
    var socket = io.connect('http://' + document.domain + ':' + location.port + '/test');
    // this is a callback that triggers when the "my response" event is emitted by the server.
    socket.on('my response', function(msg) {
        $('#log').append('<p>Received: ' + msg.data + '</p>');
    });
    //example of triggering an event on click of a form submit button
    $('form#emit').submit(function(event) {
        socket.emit('my event', {data: $('#emit_data').val()});
        return false;
    });
});
And that, effectively, is the bones of sending messages between client and server. In this specific example, we want the server to be continually working in the background generating new information, while at the same time allowing new clients to connect, and pushing new information to connected clients. For this purpose, we’ll be using the Python threading module to create a thread that generates random numbers regularly, and emits the newest value to all connected clients. Hence, in application.py, we define a thread object that will continually create random numbers and emit them using socketIO separately to the main flask process:

#Random Number Generator Thread
thread = Thread()
thread_stop_event = Event()
class RandomThread(Thread):
    def __init__(self):
        self.delay = 1
        super(RandomThread, self).__init__()
    def randomNumberGenerator(self):
        """
        Generate a random number every 1 second and emit to a socketio instance (broadcast)
        Ideally to be run in a separate thread?
        """
        #infinite loop of magical random numbers
        print("Making random numbers")
        while not thread_stop_event.isSet():
            number = round(random()*10, 3)
            print number
            socketio.emit('newnumber', {'number': number}, namespace='/test')
            sleep(self.delay)
    def run(self):
        self.randomNumberGenerator()
In our main Flask code then, we start this RandomThead running, and then catch the emitted numbers in Javascript on the client. Client presentation is done, for this example, using a simple bootstrap themed page contained in the Flask Template folder, and the number handling logic is maintained in the static JavaScript file application.js. A running list of 10 numbers is maintained and all connected clients will update simultaneously as new numbers are generated by the server.

 //Client-side Javascript code for handling random numbers
$(document).ready(function(){
    //connect to the socket server.
    var socket = io.connect('http://' + document.domain + ':' + location.port + '/test');
    var numbers_received = [];
    //receive details from server
    socket.on('newnumber', function(msg) {
        console.log("Received number" + msg.number);
        //maintain a list of ten numbers
        if (numbers_received.length >= 10){
            numbers_received.shift()
        }
        numbers_received.push(msg.number);
        numbers_string = '';
        for (var i = 0; i < numbers_received.length; i++){
            numbers_string = numbers_string + '<p>' + numbers_received[i].toString() + '</p>';
        }
        $('#log').html(numbers_string);
    });
});
And in the application.py file:

#Python code to start Asynchronous Number Generation
@app.route('/')
def index():
    #only by sending this page first will the client be connected to the socketio instance
    return render_template('index.html')
@socketio.on('connect', namespace='/test')
def test_connect():
    # need visibility of the global thread object
    global thread
    print('Client connected')
    #Start the random number generator thread only if the thread has not been started before.
    if not thread.isAlive():
        print("Starting Thread")
        thread = RandomThread()
        thread.start()
And that’s the job. Flask served web pages that react to events on the server. I was tempted to add a little graph using HighCharts or AmCharts, but I’m afraid time got the better of me. Perhaps in part2. The final output should look like this:

Flask screenshot
You can find all of the source code on GitHub, with instructions on how to install the necessary libraries etc. Feel free to adapt to your own needs, and leave any comments if you come up with something neat or have any problems.

This functionality is documented in the original documentation for Flask-SocketIO. The documentation and tutorials are quite comprehensive and worth working through if you are interested in more.
