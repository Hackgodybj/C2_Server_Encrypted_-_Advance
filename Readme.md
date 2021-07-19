# Python-C2-Server
Python C2 Server is a multi client C2/post exploitation framework with some spyware features. Powered by Python 3.8. 6 and QT Framework.

## Basic Understating 

It’s very common that after successful exploitation an attacker would put an agent that maintains communication with a c2 server on the compromised system, and the reason for that is very simple, having an agent that provides persistency over large periods and almost all the capabilities an attacker would need to perform lateral movement and other post-exploitation actions is better than having a reverse shell for example. There are a lot of free open source post-exploitation toolsets that provide this kind of capability, like ``Metasploit``, ``Empire`` and many others, and even if you only play `CTF's` it’s most likely that you have used one of those before.

This tool is still a work in progress, I finished the base but I’m still going to add more execution methods and more capabilities to the agent. After adding new features I will keep `Github Repo` similar to this one, so that people with more experience give feedback and suggest improvements, while people with less experience learn.

## About c2 servers / agents

As far as I know,

This c2 server should be able to:

    Start and stop listeners.
    Generate payloads.
    Handle agents and task them to do stuff.

An agent should be able to:

    Download and execute its tasks.
    Send results.
    Persist.

A listener should be able to:

    Handle multiple agents.
    Host files.

And all communications should be encrypted.

## Listeners

Listeners are the core functionality of the server because they provide the way of communication between the server and the agents. I decided to use http listeners, and I used flask to create the listener application.

## Flask Application

The flask application which provides all the functionality of the listener has 5 routes: /reg, /tasks/<name>, /results/<name>, /download/<name>, /sc/<name>.

## Agents

As mentioned earlier, I wrote two agents, one in powershell and the other in c++. Before going through the code of each one, let me talk about what agents do.

When an agent is executed on a system, first thing it does is get the hostname of that system then send the registration request to the server (/reg as discussed earlier).

After receiving the response which contains its name it starts an infinite loop in which it keeps checking if there are any new tasks, if there are new tasks it executes them and sends the results back to the server.

After each loop it sleeps for a specified amount of time that’s controlled by the server, the default sleep time is 3 seconds.

## C++ Agent

Sending http requests wasn’t as easy as it was in powershell, I used the winhttp library and with the help of the Microsoft documentation I created two functions, one for sending GET requests and the other for sending POST requests. And they’re almost the same function so I guess I will rewrite them to be one function later.

## Agent Handler

An Agent object is instantiated with a name, a listener name, a remote address, a hostname, a type and an encryption key.

## Payloads Generator

Any c2 server would be able to generate payloads for active listeners, as seen earlier in the agents part, we only need to change the IP address, port and key in the agent template, or just the IP address and port in case of the c++ agent.

## Powershell 

Doing this with the powershell agent is simple because a powershell script is just a text file so we just need to replace the strings REPLACE_IP, REPLACE_PORT and REPLACE_KEY.

The powershell function takes a listener name, and an output name. It grabs the needed options from the listener then it replaces the needed strings in the powershell template and saves the new file in two places, /tmp/ and the files path for the listener. After doing that it generates a download cradle that requests /sc/ (the endpoint discussed in the listeners part).

## Windows Executable (C++ Agent)

It wasn’t as easy as it was with the powershell agent, because the c++ agent would be a compiled PE executable.

It was a huge problem and I spent a lot of time trying to figure out what to do, that was when I was introduced to the idea of a stub.

The idea is to append whatever data that needs to be dynamically assigned to the executable, and design the program in a way that it reads itself and pulls out the appended information.

In the source of the agent I added a few lines of code that do the following:

    Open the file as a file stream.
    Move to the end of the file.
    Read 2 lines.
    Save the first line in the IP variable.
    Save the second line in the port variable.
    Close the file stream.

## Encryption

I’m not very good at cryptography so this part was the hardest of all. At first I wanted to use AES and do Diffie-Hellman key exchange between the server and the agent. However I found that powershell can’t deal with big integers without the .NET class BigInteger, and because I’m not sure that the class would be always available I gave up the idea and decided to hardcode the key while generating the payload because I didn’t want to risk the compatibility of the agent. I could use AES in powershell easily, however I couldn’t do the same in c++, so I decided to use a simple xor but again there were some issues, that’s why the winexe agent won’t be using any encryption until I figure out what to do. 

## Server

The AESCipher class uses the AES class from the pycrypto library, it uses AES CBC 256.

An AESCipher object is instantiated with a key, it expects the key to be base-64 encoded.

## PowerShell Agent

The powershell agent uses the .NET class System.Security.Cryptography.AesManaged.

First function is the Create-AesManagedObject which instantiates an AesManaged object using the given key and IV. It’s a must to use the same options we decided to use on the server side which are CBC mode, zeros padding and 32 bytes key length.

## Listeners / Agents Persistency

I used pickle to serialize agents and listeners and save them in databases, when you exit the server it saves all of the agent objects and listeners, then when you start it again it loads those objects again so you don’t lose your agents or listeners.

For the listeners, pickle can’t serialize objects that use threads, so instead of saving the objects themselves I created a dictionary that holds all the information of the active listeners and serialized that, the server loads that dictionary and starts the listeners again according to the options in the dictionary.

I created wrapper functions that read, write and remove objects from the databases. 

## Installation 

Just install the requirements:

`pip3 install -r requirements.txt`

Then start the server:

`python3 c2.py`
