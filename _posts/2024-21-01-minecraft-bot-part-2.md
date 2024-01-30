---
layout:     post
title:      "Creating a Minecraft Discord Integration Bot - Part 2"
subtitle:   "Creating Cogs - RCON Commands and Grafana Integration"
date:       2023-12-13 12:00:00
author:     "Dave C"
catalog: true
published: true
header-mask: 0.5
header-img: "img/in-post/post-minecraft-discord/header_bg_part2.png"
tags:
  - minecraft
  - discord
  - code
  - python
  - bots
---

# Creating our first Cog – RCON Commands

One of the reasons I love Python so much is that there is just such an active and diverse community who share their projects from so many different areas. There is usually a project which if not completely has the functionality you need, can perform some of it, or a project can serve as inspiration or a template to start your own project from scratch. 

One of the primary use cases for this bot is to be able to send commands directly from a discord channel to administer the Minecraft server. Luckily, there are several repos where most of the heavy lifting has already been done. This module looks like it will fit our purpose:  https://pypi.org/project/mcrcon/. 

### Initial Steps

Before anything, I had pagraignix enable RCON on the gameserver itself, adding the following entries to the server.properties file. 
```
enable-rcon=true
rcon.port=port_number
rcon.password= rcon_password
```

And before we get started with our cog file, install mcrcon with pip. 'Pip install mcrcon'. Make sure its installed in he same virtual environment we were working in while creating main.py.

For our new cog, let’s make a new python file in the proper location for our cogs. Firstly, lets create a new subfolder of ./cogs/ named qc_rcon_commands and create the file qc_rcon_commands.py. (./cogs/qc_rcon_commands/qc_rcon_commands.py)
Creating the file with our initial imports:

```python
import discord 
from discord import app_commands
from discord.ext import commands
from dotenv import load_dotenv
import asyncio
import os
import time 
from discord.ext.commands import has_permissions
from typing import Optional
from mcrcon import MCRcon
```

Now, straight to dealing with how to manage our RCON environment variables. I like to set the data types we would like to pull from the dot env file just so are presenting that data consistently across our codebase as we call upon the variable. 

```python
load_dotenv()

rcon_host=str(os.getenv("RCON_HOST"))
rcon_password=str(os.getenv("RCON_PASSWORD"))
rcon_port=int(os.getenv("RCON_PORT"))
```

## The Basics of a Cog

Cogs are discord extensions, which are loaded as a class containing its own attributes and methods.. in our case, we want to create a new cog, WITH all the methods available in discord.py’s commands.Cog library, so we instantiate the class as a sub class of commands.Cog: 

```python
class Quantum_RCON_Commands_Cog(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.logger = bot.logger
```

Of course we need to call __init__ since a new subclass is being created, and we are going to pass two parameters for __init__, self referring to the instance Quantum_RCON_Commands_Cog as a sub instance of commands.Cog (essentially, declaring itself as a separate entity of commands.Cog) and bot, which is the instance of our primary bot which is already created in main.py and which will be loading this cog. 
We can also use our same log handler as defined in main.py, since we are passing an instance of the bot to the cog, we are able to use its own log hander by setting attributes – self.logger = bot.logger, now Quantum_RCON_Commands_Cog and QCAdmin modules are sharing the same instance of the logger.  


## Using the RCON Module 

The MCRcon library provides a straightforward implementation of Minecraft’s Rcon protocol in Python to provide a client for handling Remote Commands (RCON) to a Minecraft server.
The recommend way to run this client is using the python ‘with’ statement. This ensures that the socket is correctly closed when you are done with it rather than being left open.
Example:

```python
from mcrcon import MCRcon
with MCRcon("10.1.1.1", "sekret") as mcr:
    resp = mcr.command("/whitelist add bob")
    print(resp)
```

While you can use it without the ‘with’ statement, you have to connect manually, and ideally disconnect. I think you'll agree - the code just doesn't read as nicely:

```python
mcr = MCRcon("10.1.1.1", "sekret")
mcr.connect()
resp = mcr.command("/whitelist add bob")
print(resp)
mcr.disconnect()	
```

### Our first Minecraft RCON command - "Say"

So like our ping command we wrote for QCAdmin which would make the bot respond in the server to a user message – Let’s make it much cooler and have the bot respond in the Minecraft server! 
Creating another slash commands group, let’s call it.. **rcon**. And then with our decorator `@rcon.command`, enter the slash command name and description. 
As most of the time with discord bots, we want to create an asynchronous function and our goal is that when the command is executed, the function say(self, *thing_to_say: str) is called. 

In the function, a command string is created using f-string formatting. It concatenates the word "say" with the thing_to_say arguments passed to the command. I really love this method of a means to submit arguments of multiple words including spaces. It is particularly useful for submitting written sentences as arguments. An asterisk before the argument will join all of the words together into one string to be passed to the function. 

```python
rcon = app_commands.Group(name="rcon", description="Send RCON commands to the Minecraft Server.")
    @rcon.command(name="say", description="Send a message from the Bot to the server. Usage <message>")
    async def say(self, *thing_to_say: str):
        """ Send a message from the Bot to the server. Usage <message>"""
        command = f"say {thing_to_say}"
        with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
            response = mcr.command(command)
            self.logger.info(f"Bot said {thing_to_say} in the server chat.")
```

Next, a context manager is used with the MCRcon object, which connects to the Minecraft RCON server. We have everything we need in our dot env files to connect to the server and perform the commands.

Within the context, the command method of the MCRcon object is called with the command string as an argument. This sends the command to the Minecraft server using the RCON protocol and retrieves the response.

So all we need to do is use our user arguments to send that string to the MCRcon command method, and in this case, we just need to pass our command concatenations as a single string: `command = f"say {thing_to_say}"`. Easy, we are just making commands to be written into the game console after all. As per the guidelines for proper gameserver socket management, we use with `MCRcon(rcon_host, rcon_password, port=rcon_port)`` to establish our connection. Then we set a variable or our response and define it as mcr.command(command) which is the method which sends our string to the server console. 

### Our Second Command - Essentials: Ping

Which server admin doesn't obsess about server client latency? Let's make a command to monitor the latency between the discord bot and the gameserver. 

I'm sure there are several modules specifically for latency monitoring but for this simple calculation, Python's native `time.time()` will take return the current time in seconds since the Epoch, which is a specific time reference in the past. To calculate the time between the rcon request and response, we can take two measures of time.time() and calculate the time in seconds between them. 

Let's call our two measures, `start_time` and `end_time`, measure their difference, and convert that number from seconds to milliseconds (ms). Latency is never presented in decimals so lets round our result by encapsulating our time difference - ```end_time - start_time``` with ```round()``` .

```python
@rcon.command(name="status", description="Check the server status.")
    async def status(self, Interaction: discord.Interaction):
        """ Check the server status."""
        try:
            start_time = time.time()
            command = f"status"
            with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
                response = mcr.command(command)
                end_time = time.time()
            # get the ping of the server, start time - end time * 1000 to get the latency in ms
            latency = round((end_time - start_time) * 1000)  
            await Interaction.response.send_message(f"Server Status: {response}\nLatency: {latency}ms")
            self.logger.info(f"Server Status: {response}\nLatency: {latency}ms")
        except Exception as e:
            await Interaction.response.send_message(f"Failed to retrieve server status: {e}")
            self.logger.error(f"Failed to retrieve server status: {e}")
```


