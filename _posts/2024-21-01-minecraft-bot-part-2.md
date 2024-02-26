---
layout:     post
title:      "Creating a Minecraft Discord Integration Bot - Part 2"
subtitle:   "Creating Cogs - RCON Commands and Grafana Integration"
date:       2024-01-30 12:00:00
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

I split this post into two parts, with part 1 just about wrapping up the creation of our custom cog loader. In Part 2, we will actually create some cogs that we can start to use in Discord. 

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

### The Basics of a Cog: The Setup Function

Finally, to make sure that our bot has instructions to load the cog we need a setup function to call the ``add_cog`` method from our bot. At the very bottom of the cog file, create an asynchronous setup function, and pass the cog name and the bot instance to it. This will run each time the cog is loaded by our bot, defined in main.py. 

```python
#discord setup function for the main bot to load the cog
async def setup(bot):
    """ Add the cog to the bot."""
    await bot.add_cog(Quantum_RCON_Commands_Cog(bot))
```

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

### Our Second Command - Essentials: Ping / Server Status

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

Now, since we have RCON permissions with our Bot, there's no limitation in the set of commands we can create for this cog which can be called from discord, but, Minecraft has a *lot* of console commands. So this is a great time to ask a few questions to the primary stakeholders / users of the bot to determine some baseline / day one command set to aim for. 

The commands reference will be essential to account for the appropriate command parameters, so we will keep this open as we write our cog commands.

https://minecraft.fandom.com/wiki/Commands - Minecraft command reference

After speaking with some players on the server, it was clear that the most important commands would be to set the server time of day and to set the weather, since in ATM9 bad things happen at night and monsters had been ganking new players as soon as night fell (To which I became a victim too in spite of Ten gifting me weapons and armor).

Let's start with setting the gameserver weather, and let's check the Command reference: 

``weather (clear|rain|thunder) [<duration>]``

#### Arguments

``clear|rain|thunder``

``clear – Set the weather to clear.``
``rain – Set the weather to rain (or snowfall in cold biomes).``
``thunder – Set the weather to a thunderstorm (or blizzard in cold biomes).``

So this one is really simple, we just need to create a discord interaction that will call /weather in rcon and pass an argument for the weather type along with the interaction. We will create this command under our already created rcon commands group, by adding the decorator ``@rcon.command`` to the start of the function. 

As always we pass self (an instance of the rcon_commands cog including the bot as an attribute) and since these are interactions and not basic discord text commands, we use Interaction: discord.Interaction before adding a string type argument for the weather type. 

Its always a good idea to do some input validation for server commands. As a basic check, I will check the supplied arguments to make sure that they match the possible weather states as listed in the command reference. Let's also add some logic about how to inform the user and proceed if they do enter an invalid weather type. Since we passed self to the command, this means we can use our existing log handler, so let's make sure we are capturing steps as we progress through the command. It wouldn't hurt to add some try and except blocks to this code but this will do the job: 

```python
@rcon.command(name="weather", description="Change the weather. Usage <weather_type> \n Valid weather types: clear, rain, thunder")
@has_permissions(manage_channels=True)
async def weather(self, Interaction: discord.Interaction, weather_type: str):
    """ Change the weather. Usage <weather_type> \n Valid weather types: clear, rain, thunder"""
    valid_types = ["clear", "rain", "thunder"]
    if weather_type.lower() not in valid_types:
        await Interaction.response.send_message("Invalid weather type. Choose from clear, rain, or thunder.")
        self.logger.warning(f"{Interaction.user} attempted to set an Invalid weather type: {weather_type}")
        return
    command = f"/weather {weather_type}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Weather changed to {weather_type}.")
        self.logger.info(f"{Interaction.user} changed Weather to {weather_type}.")
```

And let's give it a test

![Alt text](/img/in-post/post-minecraft-discord/weather_thunder.png)

![Alt text](/img/in-post/post-minecraft-discord/weather_clear.png)

Awesome! Seeing the command entered in Discord and having it execute in the game - there's just something special about that which makes projects like this so rewarding. 

The next problem for the server was making sure players ended up in the same place and in ATM9 there are a lot of threats. So a teleport command would be ideal to port new players to the established settlement. Unforunately for me, I decided to do the trek from the spawn to the player town solo and died several times along the way. I've never been one to do things the easy way - I'll just say it's part of The Art of Failing. Anyway, let's get to our teleport command. 

#### Teleport Commands

After reading through the Minecraft commands documentation I was impressed with the sheer amount of optional parameters available. The teleport command can accept multiple targets and the player rotation can optionally be set to face a specific location or entity. In most cases, I won't be needing every kind of teleport and for our server, teleport <target> <destinationplayer> would suffice. I decided to streamline several commands to keep the functionality we want without having to account for all of the optional parameters, but as an example, let's account for all acceptable teleport commands in our teleport command.  

- ``teleport <destination>``
- ``teleport <targets> <destination>``: Teleports the executor or the specified entity(s) to the position of an entity, and makes its rotation the same as the specified entity's.
- ``teleport <location>``: Teleports the executor to a certain position (and changes its rotation to the command's execution rotation).
- ``teleport <targets> <location>``: Teleports the entity(s) to a certain position (without changing their rotation).
- ``teleport <targets> <location> <rotation>``
- ``teleport <targets> <location> facing <facingLocation>``
- ``teleport <targets> <location> facing entity <facingEntity> [<facingAnchor>]``: Teleports the entity(s) to a certain position and changes their rotation to the specified rotation.

Let's start with a basic person to person teleport which requires two arguments, target and destination. Target, being the player to be teleported and Destination, which is the position of the player the target will be teleported to. 

```python
@rcon.command(name="teleport", description="Teleport a player. Usage <player> <destinationplayer>")
@has_permissions(manage_channels=True)
async def teleport(self, Interaction: discord.Interaction, player: str, dest_player: str):
    """ Teleport a player. Usage <player> <destinationplayer>"""
    command = f"tp {player} {dest_player}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Teleported {player} to {dest_player}.")
```

Now that I had a basic idea of how to create a discord interaction that would pass a command to the RCON module, I was able to work through the Minecraft server command documentation and create a bunch more commands that could be executed from discord. I won't go through each one in detail since I'm simply creating a new command each time with the available parameters from the documentation. In the end, I was able to build an extensive command list allowing for discord based administration of a Minecraft gameserver.  

### Total Commands List

* **Teleport:** Teleport one player to another. 
* **Weather:** Set the server weather. 
* **Ability:** Set player ability values.
* **Advancement:** Grant or Revoke player advancements.
* **Ban:** Ban a player from the server.
* **Ban IP:** Ban an IP address from the server. 
* **Ban List:** Get a full list of banned players. 
* **Clear:** Clear items from a players inventory.
* **Clone:** Clone blocks from one location to another. 
* **Damage:** Damage a specific server entity. 
* **Daylock:** Lock the server day/night cycle. 
* **Difficulty:** Set the game server environment difficulty.
* **Gamerule:** Set a game rule.
* **Effect:** Grant an effect to a player or entity.
* **Enchantment:** Place an Enchantment upon an item.
* **Give:** Spawn an object into a players inventory.
* **Kick:** Kick a player from the server.
* **Listplayers:** List the players currently connected to the server. 
* **op:** Grant operator status to a player. 
* **setidletimeout:** Set Idle Time before players are kicked
* **setmaxplayers:** Set the maximum number of players allowed to connect
* **summon:** Summon a creature at the target location
* **fill:** Fill region with a specific block. 
* **fillbiome:** Fill region with a specific biome. 
* **place:** Place an object at a specific coordinate, with an angle and rotation. 
* **seed:** Get the current world seed. 
* **setblock:** Place a specific block at a target location. 
* **setworldspawn:** Set the default world spawn for new players. 
* **setspawnpoint:** Set the spawn point coordinates for a player
* **time:** Get the current server time. 


#### Full Breakdown of Commands

```python
@rcon.command(name="ablity" , description="Set a player's ability value. Usage <player> <ability> <value>")
@has_permissions(manage_channels=True)
async def ability(self, Interaction: discord.Interaction, player: str, ability: str, value: int):
    """ Set a player's ability value. Usage <player> <ability> <value>"""
    command = f"{player} {ability} {value}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"{player} ability: {ability} set to {value}.") 
        self.logger.info(f"{Interaction.user} set {player} ability: {ability} to {value}.")
```

```python
@rcon.command(name="advancement" , description="Grant or revoke advancements to players. Usage <player> <action> <advancement>")
@has_permissions(manage_channels=True)
async def advancement(self, Interaction: discord.Interaction, player: str, action: str, advancement: str):
    """ Grant or revoke advancements to players. Usage <player> <action> <advancement>"""
    command = f"{player} {action} {advancement}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"{player} was {action} {advancement}.") 
        self.logger.info(f"{Interaction.user} {player} {action} {advancement}.")
```


```python
@rcon.command(name="ban", description="Ban a player from the server. Usage <player>")
@has_permissions(manage_channels=True)
async def ban(self, Interaction: discord.Interaction, player: str):
    """ Ban a player from the server. Usage <player>"""
    command = f"ban {player}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"{player} has been banned from the server.")
        self.logger.info(f"{Interaction.user} banned {player}.")
```

```python
@rcon.command(name="ban-ip", description="Ban an IP address from the server. Usage <ip>")
@has_permissions(manage_channels=True)
async def ban_ip(self, Interaction: discord.Interaction, ip: str):
    """ Ban an IP address from the server. Usage <ip>"""
    command = f"ban-ip {ip}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"{ip} has been banned from the server.")
        self.logger.info(f"{Interaction.user} IP banned {ip}.")
```

```python
@rcon.command(name="banlist", description="List all banned players.")
@has_permissions(manage_channels=True)
async def banlist(self, Interaction: discord.Interaction):
    """ List all banned players."""
    command = "banlist"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        if response:
            await Interaction.response.send_message(f"Banned players: {response}")
        else:
            await Interaction.response.send_message("No players are banned.")
        self.logger.info(f"{Interaction.user} listed banned players.")
```

```python
@rcon.command(name="clear", description="Clear items from a player's inventory. Usage <player> [item] [count]")
@has_permissions(manage_channels=True)
async def clear(self, Interaction: discord.Interaction, player: str, item: str=None, count: int=None):
    """ Clear items from a player's inventory. Usage <player> [item] [count]"""
    command = f"clear {player} {item if item else ''} {count if count else ''}".strip()
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Cleared items from {player}'s inventory.")
        self.logger.info(f'{Interaction.user} Cleared items from {player} inventory.')
```

```python
@rcon.command(name="clone", description="Clone blocks. Usage <start_pos> <end_pos> <destination> [mask_mode] [clone_mode] [tile_mode]")
@has_permissions(manage_channels=True)
async def clone(self, Interaction: discord.Interaction, start_pos: int, end_pos: int, destination: int, mask_mode: bool=None, clone_mode: bool=None, tile_mode: bool=None):
    """ Clone blocks. Usage <start_pos> <end_pos> <destination> [mask_mode] [clone_mode] [tile_mode]"""
    command = f"clone {start_pos} {end_pos} {destination} {mask_mode if mask_mode else ''} {clone_mode if clone_mode else ''} {tile_mode if tile_mode else ''}".strip()
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
    await Interaction.response.send_message(f"Blocks cloned from {start_pos} to {end_pos} to {destination}." + (f" with mask mode {mask_mode}" if mask_mode else "") + (f", clone mode {clone_mode}" if clone_mode else "") + (f", tile mode {tile_mode}" if tile_mode else "") + ".")
```

```python
@rcon.command(name="damage", description="Damage entities. Usage <entities> <amount>")
async def damage(self, Interaction: discord.Interaction, entities: str, amount: int):
    """ Damage entities. Usage <entities> <amount>"""
    command = f"damage {entities} {amount}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
    await Interaction.response.send_message(f"Damaged {entities} by {amount}.")
```

```python
@rcon.command(name="daylock", description="Lock or unlock the day-night cycle. Alias: alwaysday. Usage <action>")
@has_permissions(manage_channels=True)
async def daylock(self, Interaction: discord.Interaction, action: str):
    """ Lock or unlock the day-night cycle. Alias: alwaysday. Usage <action>"""
    command = f"daylock {action}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
    await Interaction.response.send_message(f"Daylock {action}.")
```

```python
@rcon.command(name="difficulty", description="Change the game difficulty. Usage <level>")
@has_permissions(manage_channels=True)
async def difficulty(self, Interaction: discord.Interaction, level: int):
    """ Change the game difficulty. Usage <level>"""
    command = f"difficulty {level}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Game difficulty set to {level}.")
```
```python        
@rcon.command(name="gamerule", description="Set or query a game rule value. Usage <rule> [value]")
@has_permissions(manage_channels=True)
async def gamerule(self, Interaction: discord.Interaction, rule: str, value: str=None):
    """ Set or query a game rule value. Usage <rule> [value]"""
    command = f"gamerule {rule} {value if value else ''}".strip()
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
    await Interaction.response.send_message(f"Game rule {rule} set to {value}." if value else f"Game rule {rule} is {response}.")
```

```python
@rcon.command(name="effect", description="Give an effect to a player or entity. Usage <target> <effect> [duration] [amplifier]")
@has_permissions(manage_channels=True)
async def effect(self, Interaction: discord.Interaction, target: str, effect: str, duration: int=None, amplifier: str=None):
    """ Give an effect to a player or entity. Usage <target> <effect> [duration] [amplifier]"""
    command = f"effect give {target} {effect} {duration if duration else ''} {amplifier if amplifier else ''}".strip()
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Effect {effect} given to {target}.")
```

```python
@rcon.command(name="enchantment", description="Enchant a player item. Usage <player> <enchantment> [level]")
@has_permissions(manage_channels=True)
async def enchant(self, Interaction: discord.Interaction, player: str, enchantment: str, level: int= None):
    """ Enchant a player item. Usage <player> <enchantment> [level]"""
    command = f"enchant {player} {enchantment} {level if level else ''}".strip()
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Enchantment {enchantment} applied to {player}.")
```

```python
@rcon.command(name="give", description="Give items to a player. Usage <player> <item> <amount>")
@has_permissions(manage_channels=True)
async def give(self, Interaction: discord.Interaction, player: str, item: str, amount: int):
    """ Give items to a player. Usage <player> <item> <amount>"""
    command = f"give {player} {item} {amount}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Gave {amount} of {item} to {player}.")
```

```python
@rcon.command(name="kick", description="Kick a player from the server. Usage <player> [reason]")
@has_permissions(manage_channels=True)
async def kick(self, Interaction: discord.Interaction, player: str,  *, reason: str=None):
    """ Kick a player from the server. Usage <player> [reason]"""
    command = f"kick {player} {reason}" if reason else f"kick {player}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"{player} has been kicked from the server. Reason: {reason}" if reason else f"{player} has been kicked from the server.")
```

```python
@rcon.command(name="listplayers", description="List all players on the server.")
@has_permissions(manage_channels=True)
async def list_players(self, Interaction: discord.Interaction):
    """ List all players on the server."""
    command = "list"
    with MCRcon(rcon_host, rcon_password, port=int(rcon_port)) as mcr:  # Ensure port is an integer
        response = mcr.command(command)
        await Interaction.response.send_message(f"Denizens on the server: {response}")
```

```python
@rcon.command(name="op", description="Grant operator status to a player. Usage <player>")
@has_permissions(manage_channels=True)
async def op(self, Interaction: discord.Interaction, player: str):
    """ Grant operator status to a player. Usage <player>"""
    command = "op"
    async with MCRcon(rcon_host, rcon_password, port=int(rcon_port)) as mcr:  # Ensure port is an integer
        response = mcr.command(command)
        await Interaction.response.send_message(f"Operator status granted to {player},  {response}")
```

```python
@rcon.command(name="setidletimeout", description="Set the idle timeout for players. Usage <timeout>")
@has_permissions(manage_channels=True)
async def setidletimeout(self, Interaction: discord.Interaction, timeout: int):
    """ Set the idle timeout for players. Usage <timeout>"""
    command = f"setidletimeout {timeout}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Idle timeout set to {timeout} minutes.")
```

```python
@rcon.command(name="setmaxplayers", description="Set the maximum number of players. Usage <max_players>")
@has_permissions(manage_channels=True)
async def setmaxplayers(self, Interaction: discord.Interaction, max_players: int):
    """ Set the maximum number of players. Usage <max_players>"""
    command = f"setmaxplayers {max_players}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Maximum players set to {max_players}.")
```

```python 
@rcon.command(name="summon", description="Summon an entity. Usage <entity> <x> <y> <z>")
@has_permissions(manage_channels=True)
async def summon(self, Interaction: discord.Interaction, entity: str, x: int, y: int, z: int):
    """ Summon an entity. Usage <entity> <x> <y> <z>"""
    command = f"summon {entity} {x} {y} {z}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Summoned {entity} at ({x}, {y}, {z}).")
```


#### Running into Limits with Slash Commands Groups

I was surprised at this point when I attempted to test my next rcon command, the commands group failed to synchronize with the Discord API due to a exceeding the Slash Commands limit. I hadn't run into this before in the past, but I also hadn't created a commands group of this size. After a couple of minutes of Googling, I found the information around slash command limits in the slash commands official documentation : https://discord.com/developers/docs/interactions/application-commands .

At the time of writing, slash commands groups are limited to 8000 characters. I'm not sure as to why this is the case, perhaps to improve latency and user experience with the Discord API or to just limit the amount of storage required for the registration/sync of slash commands, but it doesn't seem like they will be increasing this limit in the near future. 

#### Creating a World Command Group

The most straightforward way to address this limitation is just by creating a second slash commands group. By taking a look at the types of commands already implemented and still pending, I decided to categorize the commands into two groups, @rcon.command and @world.command, the world commands group containing the commands to affect the overall game world and rcon commands to affect the player and server itself. 

```python  
world = app_commands.Group(name="world", description="Send RCON commands to the Minecraft Server - Manage World Attributes.")
@world.command(name="fill", description="Fill a region with a specific block. Usage <start_pos> <end_pos> <block> [mode]")
async def fill(self, Interaction: discord.Interaction, start_pos: int, end_pos: int, block: str, mode: str=None):
    """ Fill a region with a specific block. Usage <start_pos> <end_pos> <block> [mode]"""
    command = f"fill {start_pos} {end_pos} {block} {mode if mode else ''}".strip()
    async with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Filled region from {start_pos} to {end_pos} with {block}." + (f" in mode {mode}" if mode else "") + ".")
```
```python
@world.command(name="fillbiome", description="Fill a region with a specific biome. Usage <start_pos> <end_pos> <biome>")
@has_permissions(manage_channels=True)
async def fillbiome(self, Interaction: discord.Interaction, start_pos: int, end_pos: int, biome: str):
    """ Fill a region with a specific biome. Usage <start_pos> <end_pos> <biome>"""
    command = f"fillbiome {start_pos} {end_pos} {biome}"
    async with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Filled region from {start_pos} to {end_pos} with {biome}.")
```
```python
@world.command(name="place", description="Usage <feature> <x> <y> <z> [rotation] [mirror] [mode]")
@has_permissions(manage_channels=True)
async def place(self, Interaction: discord.Interaction, feature: str, x: int, y: int, z: int, rotation: int=None, mirror: bool=None, mode: str=None):
    """ Place a feature at a location. Usage <feature> <x> <y> <z> [rotation] [mirror] [mode]"""
    command = f"setblock {x} {y} {z} {feature}"
    if rotation:
        command += f" rotation={rotation}"
    if mirror:
        command += f" mirror={mirror}"
    if mode:
        command += f" mode={mode}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Placed {feature} at ({x}, {y}, {z})" + (f" with rotation {rotation}" if rotation else "") + (f", mirror {mirror}" if mirror else "") + (f", in mode {mode}" if mode else "") + ".")
```

```python
@world.command(name="seed", description="Get the world seed.")
async def seed(self, Interaction: discord.Interaction):
    """ Get the world seed."""
    command = "seed"
    with MCRcon(rcon_host, rcon_password, port=int(rcon_port)) as mcr:  # Ensure port is an integer
        response = mcr.command(command)
        await Interaction.response.send_message(f"World seed: {response}")
```
```python
@world.command(name="setblock", description="Place a block at a location. Usage <x> <y> <z> <block> [mode]")
@has_permissions(manage_channels=True)
async def setblock(self, Interaction: discord.Interaction, x: int, y: int, z: int, block: str, mode:str=None):
    """ Place a block at a location. Usage <x> <y> <z> <block> [mode]"""
    command = f"setblock {x} {y} {z} {block}" + (f" {mode}" if mode else "")
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"Block {block} placed at ({x}, {y}, {z})" + (f" in mode {mode}" if mode else "") + ".")
```
```python
@world.command(name="setworldspawn", description="Set the world spawn. Usage [x y z]")
@has_permissions(manage_channels=True)
async def setworldspawn(self, Interaction: discord.Interaction, x: int, y: int, z: int):
    """ Set the world spawn. Usage [x y z]"""
    command = f"setworldspawn {x} {y} {z}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
        await Interaction.response.send_message(f"World spawn set to ({x}, {y}, {z}).")
```
```python
@world.command(name="setspawnpoint", description="Set the world spawn. Usage [x y z]")
async def spawnpoint(self, Interaction: discord.Interaction, player: str,  pos: str=None):
    """ Set the world spawn. Usage [x y z]"""
    command = f"spawnpoint {player} {pos}"
    with MCRcon(rcon_host, rcon_password, port=rcon_port) as mcr:
        response = mcr.command(command)
    await Interaction.response.send_message(f"Spawnpoint set to {pos} for {player}.")
```
```python
@world.command(name="time", description="Set or query the world time. Usage <action> [value]")
@has_permissions(manage_channels=True)
async def time(self, Interaction: discord.Interaction, action: str, value: Optional[int] = None):
    """ Set or query the world time. Usage <action> [value]"""
    if action.lower() == "set":
        if value is not None:
            response = await self.rcon_command(f"time set {value}")
            await Interaction.response.send_message(f"Time set to {value}. Server response: {response}")
        else:
            await Interaction.response.send_message("You need to provide a value for 'set' action.")
    elif action.lower() == "query":
        response = await self.rcon_command("time query daytime")
        await Interaction.response.send_message(f"Current time: {response}")
    else:
        await Interaction.response.send_message("Invalid action. Use 'set' or 'query'.")
```

### Completing the Cog

To finish up the cog, as with our management cog we built in part 1, we will create a setup function to pass the bot instance to the cog, making sure that our slash commands are registered with the bot. 

```python
async def setup(bot):
    """ Add the cog to the bot."""
    await bot.add_cog(Quantum_RCON_Commands_Cog(bot))
```

Now we have our cog loader and our first cog! qc_rcon_commands! In this blog we went over the start to finish process of creating a custom cog, how to create slash commands - specifically, slash commands which interacted with the minecraft server using the MCRCON module, and overcame obstacles encountered due to size limits of slash command registration. 

In part 3, the finale of this series, we will build our snazziest cog, which will integrate grafana dashboard and panels into discord, allowing discord based monitoring of the game server. 