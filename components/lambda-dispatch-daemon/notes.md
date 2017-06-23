This file contains thoughts on the lambda-dispatch-daemon (LDD).


### Overview

The LDD is the intermediary between the message bus and the lambda
functions that will process the message. It is a C++ program using the
Solace C API. It will create sessions to all message VPNs it needs to
service and will create flows to the queues within those VPNs that it
is in charge of.

#### Steady-state Datapath Sequence

1. Message is received from message bus

2. Flow callback is called with the message

3. The flow and topic in the message are used to identify the lambda function
   information: which container must be run and is it currently running

4. If the container for that function is not running, create the container using
   the docker REST interface (specifying a port mapping).
   This is done via a volume mounted UNIX socket allowing sibling containers
   to be created. When the container is running, it will immediately listen on its port

5. The LDD will connect to the appropriate container over TCP and send the full message. The
   message will contain the topic, binary attachment and possibly some other
   meta data. The message is sent in proprietary format.

6. The lambda function wrapper in the container will receive the message, parse it and
   then call the user's inner lambda function, passing the topic, message data and other
   meta data via a standardized function call appropriate for that language

7. The lambda function will execute and return one of a standardized set of
   return codes.

8. While executing, the lambda function can send messages back to the message bus by calling a
   well-known function that lives in the wrapper.

9. If the lambda function returns an error, then the container is killed and no
   response is sent to the LDD. If this happens, then the LDD will close and re-establish
   its flow to the message bus, allowing that message to be attempted on another LDD.
   TBD - possibly discard the message if it has redelivered flag set...

10. If the lambda function is successful, then a success code is returned to
    the LDD, which in turn ACKs the message on the message bus

Other notable items:

1. After execution, the lambda function container will remain in place for a
   configurable amount of time. When that inactivity time has been exceeded,
   the root process of the container will die and the LDD will note that it
   is no longer in place.

   TBD - perhaps the LDD should do the inactivity timer
   for all containers. Two good reasons for this: 1. factors out this code from the lambda
   wrapper, which has to be written for lots of languages; 2. probably simplifies
   case where the container kills itself just as the LDD is trying to send it
   a message.

2. The LDD will maintain a connection to the lambda function container as long as
   it can. If the container is running when the next message is received, then the
   LDD will simply forward that message to the container.

3. If the LDD terminates the persistent connection to the container, the container's
   root process will simply die, killing the container.

