---
layout:     post
title:      "Bullshido - A Discord Game of Epic Combat"
subtitle:   "A cog to drive user engagement based on the awesome but whacky world of Martial Arts"
date:       2024-12-03 12:00:00
author:     "Dave C"
catalog: true
published: true
header-mask: 0.5
header-img: "img/in-post/bullshido/bullshido-bg.png"
tags:
  - fun
  - discord
  - code
  - games
  - python
  - bots
---

# *Foreword*

Yes, it's been 9 months since my last blog post, but I promise—it wasn’t procrastination. It was a side quest. 

You know how it goes: you start working on one thing (in this case, the "Bullshido" cog), and before you know it, the project becomes much bigger than you anticipated. Even worse, you've written a ton of code without documentation.

Then it became apparent that the project was just too large to put coherently into one post, so I decided to split it into multiple parts.

# Introduction

Today I'm excited to blog about something more light hearted, a fun discord cog which I have been working on over the past couple of months. Before diving into the technical details including a full breakdown of the code, I'd like to talk about the inspiration behind this project and what led me to create this whacky discord cog I named ***Bullshido***. 

I help to maintain and operate a discord server for a small group of friends and family, and I've always been interested in creating a bot that could help manage the server and provide some fun and engaging features. Of the many games I've created as cogs which have ended up used by so many other discord bots from around the web (who can load the cogs directly from my repo), the cog creation almost always begins on my own bot - ScrapGPT. [https://github.com/The-Sanctuary-Discord/ScrapGPT](https://github.com/The-Sanctuary-Discord/ScrapGPT)
.

Growing up in the 80s and 90s, I was captivated by the explosion of martial arts in the Americas. This era was significantly influenced by action movies and media that portrayed martial arts as a powerful, almost mystical discipline. Icons like Bruce Lee, Jean-Claude Van Damme, and Chuck Norris brought martial arts into mainstream culture, sparking a widespread interest that saw dojos popping up in every neighborhood. The esoteric nature of martial arts, combined with the mystique of the Far East, created a culture that was ripe for exploitation, with many charlatans and frauds claiming to be masters of ancient, forbidden or secret techniques, bestowed up them by a mysterious and unknown master. Obviously these techniques could be learned - for a fee, and many of these characters made millions of dollars during this period which we can call the Golden Age of Martial Arts.

![Alt text](/img/in-post/bullshido/martialmovies.png)

Without a doubt, the most infamous of these legendary martial arts charlatans was *Frank Dux*, a guy who claimed to have won the ***Kumite—a*** secret, underground, *totally-not-made-up* martial arts tournament held in Barbados. You know, the kind of tournament where the invitation probably comes in the form of a mysterious, ancient scroll delivered by a ninja at 3 a.m. Dux’s tall tale was so wild, Hollywood ate it up and spat out *Bloodsport* in 1988, with Jean-Claude Van Damme kicking his way to stardom.

Thanks to the internet, which has been the most horrible invention for guys like Frank Dux since polygraph tests— the truth about these "legendary" fighters has been dragged into the light. But the mystique of the Kumite is like a bad kung-fu movie—it just won’t quit. People still talk about it, and the legend of the Kumite lives on in the hearts of martial arts fans everywhere.

I wanted to create a discord game that captured the spirit of these martial arts movies, with their over-the-top characters, dramatic fight scenes, and larger-than-life personalities. I wanted to create a game that would allow users to create their own martial arts characters, train them, and compete in a virtual Kumite tournament against other players on the same server or globally. 

And of course, the main objective is to have fun and learn a bunch of new Python skills along the way. 

I had initially planned to cover the entire project in one post, but as I started writing, I realized that there was a lot to cover. So, I've decided to split the project into two parts. In this first part, I'll cover the high-level objectives, the cog structure, and the core components of the game, including player and guild configurations, the fighting game flow, and the strike and injury mechanics. In the second part, I'll dive into the user interactions, the fight outcomes, the AI-generated fight hype, and the game's presentation and aesthetics.

# Project Overview

## Objectives

* **Engage and Entertain Users**
  
    * **Objective:** Provide an interactive and entertaining experience for Discord users through a simulated fighting game.
    * **Description:** The cog aims to create an engaging and fun discord game which captures the over the top and whacky nature of martial arts movies from the 80s and 90s.
  
* **Implement a Robust Game System**
  
    * **Objective:** Develop a structured and balanced turn-based fighting game with comprehensive rules and mechanics.
    * **Description:** The cog includes detailed game mechanics, such as player statistics, health, stamina, and fighting styles. It uses probabilistic models to determine fight outcomes, and offers a variety of actions and results to keep gameplay dynamic and fair.

* **Promote User Interaction and Customization**
  
    * **Objective:** Encourage frequent user interactions with the cog, via personalization through in-game choices, activities and progression.
    * **Description:** Users can select fighting styles, improve their characters' stats, and compete in different fight formats. The game tracks each user’s performance, allowing for character development and stat customization, which enhances the sense of personal investment and progression. Players will be required to interact with the cog daily to maintain their character's stats and progress.
   
* **Create a Community-Driven Experience**

    * **Objective:** Foster a sense of community and friendly competition among Discord server members by allowing them to compete, bet on fights, take part in tournaments and build a fighting legacy for each user. 
    * **Description:** The cog supports multiplayer interaction, enabling users to challenge each other publicly. This helps build a community around the game, where users can participate in or spectate fights, encouraging social interaction and a competitive spirit.

* **Provide Configurable and Scalable Features**

    * **Objective:** Design the cog to be easily configurable and adaptable to different server environments and preferences. 
    * **Description:** The cog includes various configuration options for server admins, such as setting the number of rounds, stat bonuses and other key game parameters. This allows server owners to customize the game to suit their community's preferences and playstyle.

* **90's Martial Arts Movie Aesthetic**

    * **Objective:** The game presentation should give the players a sense that they are taking part in a Bloodsport like tournament. 
    * **Description:** The aesthetic should match the cheesy, over the top style of 90's martial arts movies, with dramatic fight descriptions, colorful character names, and humorous combat outcomes. The game should evoke the spirit of the era, with a mix of nostalgia and parody. I was thinking Street Fighter meets Mortal Kombat meets Bloodsport.


## Building Cogs with Red

![Alt text](/img/in-post/bullshido/cog_libraries.png)

In my previous cog writeup for Quantumcraft, I had built the bot and cog loader from scratch. This is the right approach for a purpose-built bot, where additional features are not required. I did build a custom cog loader which facilitated a modular, cog based development approach for future enhancements but as far as cog loaders go, it was as basic as they come. 

For this project, I wanted to use a more robust and feature-rich bot framework that would allow me to focus on the game mechanics and user experience, rather than the underlying bot infrastructure. I chose to use Red, a popular open-source bot framework built on top of discord.py. Red provides a wide range of features and utilities that make it easy to create complex and interactive bots, including a built-in cog system, a powerful command framework, and extensive customization options. I've been building cogs on Red for years now and the library is still under active development [https://github.com/Cog-Creators/Red-DiscordBot]. 

The main reason I keep coming back to Red is the configuration system for managing user data and guild-specific configurations. Red's built in ```Config``` object simplifies the storage and retrieval of user-specific and guild-specific data, making it an excellent choice for managing the user and guild data required when it comes to creating more complex and engaging cogs. 

Additionally, Red's built in server economy through the ```bank``` module makes it really easy to hook up cogs to the server economy and implement features such as betting, prize money and in-game purchases.


## Cog Components

![Alt text](/img/in-post/bullshido/cog_components.png)

I jotted down the high level requirements for the cog to make sure I was covering all the bases:

* **User Interactions:**

  * **UI Buttons:** I want the players to be able to use Discord UI components instead of commands to manage their characters and participate in fights. This will make the game easier to play and more engaging for users who may not be familiar with Discord bots or command-based games. Ideally I will create a seperate module for the UI components and have the main cog call the UI components as needed. In Discord.py, Views are used to create UI components which can be interacted to via button callbacks.
  
  * **Context-Aware/User Specific Validation:** Each interaction is tied to a specific user, ensuring that only the right person can click a button or make a decision for their character. Fortunately, ctx is an attribute of the discord.py library which is passed to each command, so we can use this to ensure that the user is the one who initiated the interaction.

  * **Slash-Commands:** In addition to the UI components I will also need to create slash commands for the game, which will allow users to interact with the bot using text commands. This will provide an alternative way to play the game and also be useful for admin commands. A nice feature of Redbot is that it has a built in hybrid command system which allows for both slash commands and traditional text commands to be used in the same cog.

* **Data Persistence:**

    * **Game Configuration:** Each server will have it's own customizable game config with several parameters to customize the game experience. This will include things like the number of rounds in a fight, the amount of health and stamina each character has by default, and the stat bonuses for each fighting style. Red's Config object will be used to store and retrieve this data, since it's built in and easy to use.

    * **Player Stats:** Player stats will be stored in the user data, which is also managed by Red's Config object. This will include things like the player's level, health, stamina, fighting style, and other relevant information. The player stats will be updated as the player progresses through the game, and will be used to determine the outcome of fights and other interactions. 

    * **Fighter Record:** Each player will have a record of their fights, wins, losses, and other relevant information. This data will be stored in the user data and will be used to establish rankings, betting odds, and other game features.


## Cog Structure

![Alt text](/img/in-post/bullshido/cog_structure.png)

As always, this project ended up being more complex than I initially anticipated, so I decided to break it down into several smaller helpers to make it more manageable. 

When building a cog, it's important to keep the code organized and modular, so that it's easy to maintain and extend in the future. I like to create one primary cog which handles commands, user/guild configurations, environment variables, logging, etc, and importantly, orchestration of the game flow. The helper functions are imported to the main cog and called as needed to generate objects throughout the game.

I ended up with five core files, each responsible for different aspects of the game's logic, user interaction, and mechanics.

### bullshido.py: The Core Cog

`bullshido.py` is the heart of the cog. It handles:

  * **Cog Initialization:** When the cog is loaded, bullshido.py sets up the configuration, registers default values for users and guilds, and ensures the bot is ready to receive commands.

  * **Cog Configuration:** Keeping the configuration data for the players and the server. 

  * **User Interactions and Commands:** It defines the commands players use to interact with the game. For example, users can challenge each other to fights, select their fighting styles, or check their stats.

  * **Admin or Mod Commands:** Administrative commands, such as enabling/disabling socialized medicine or setting the number of fight rounds, are also handled here.

This is the core cog for the game, including all of the user and admin slash commands and interactions, as well as the code to manage the guild and user configs. It communicates with all other files—importing constants from fighting_constants.py, receiving user inputs from ui_elements.py, managing game flow through fighting_game.py, reading and writing to the guild or user Redbot Configs and generating hype via bullshido_ai.py.


### fighting_game.py: Game Flow and Mechanics

This module encapsulates all of the gameplay mechanics and, for each game session, loops through the configured number of fight rounds and player turns until a fight outcome is achieved. It pulls strike data from fighting_constants.py into functions to calculate strike damage, hit chance, critical chance, KO chance, TKO chance, or any other statistical in-game calculations. The module communicates with bullshido.py by passing bullshido.py, which defines the main cog class Bullshido, to this module as a dependency so that fighting_game.py will have access to Cog configuration data without having to rewrite all the code to access the Redbot Configs.

For now, I am also placing my code to generate fight images in this module, which isn't exactly ideal. Ideally, the image generation logic should be decoupled and placed into a separate utility or visualization module to maintain clean separation of concerns. This would allow the core gameplay mechanics to focus solely on managing the fight logic while another module handles image generation, making future maintenance and enhancements easier. 

*   **Purpose:** This is where the game logic and mechanics are managed. It contains:
    * **Game Initialization:** It sets up the game state, pulling player data, initializing rounds, stamina, and health.
  
    * **Game Flow:** It manages turn-by-turn combat, determining hit/miss probabilities, strike outcomes, and critical hits.
  
    * **Player Stat Management:** It updates player stats like wins, losses, stamina, health and other stats as the game progresses.
  
    * **Strike Calculations and Utility Functions:** The actual damage calculations and fight outcomes, like knockouts or technical knockouts (TKO), are handled here. It also generates fight images using avatars.

    * **Fight Scoring and Outcome Determination:** Code for scoring the fight or handling TKO/KO. Updates player stats accordingly.


### fighting_constants.py: Game Constants and Data

This module is referenced by both bullshido.py and fighting_game.py to get predefined fight data for calculations and fight presentation. It's a central repository for all constants and data used in the game, which I had spent just as much time coming up with as I did writing the code for the game. 

*   **Purpose:** This file contains predefined data used across the game:
    * **Strikes Dictionary:** Lists of strikes, organized by fighting style, with associated damage ranges.

    * **Body Parts:** Possible body parts that could be injured during a fight.

    * **Injuries:** A list of potential injuries that can occur depending on the body part targeted.

    * **Fight Event Flavor:** Any constants that affect the way fights are presented visually and narratively.


### bullshido_ai.py: AI-Generated Hype and Fight Narratives

This component connects to both bullshido.py and fighting_game.py to pull player data, create a narrative, and then output it to the Discord channel.

* **Purpose:** This file uses OpenAI's API to generate dynamic fight hype and narratives. It includes:

  * **Generate Hype:** Before a fight, the bot can generate exciting, humorous descriptions of the upcoming battle between two users.

  * **Generate Narrative:** During or after a fight, it can create a play-by-play commentary to enhance the game's storytelling element.

### Player Stats, Configurations

I decided to use the tried and true player stats system from most fighting games, Health and Stamina, which are the two primary stats that determine a player's ability to fight. 

* **Primary Stats:**
  
  * **Health:** Health represents the player's overall well-being and ability to take damage. When a player's health reaches zero, they are knocked out and they lose the fight. 
  * **Stamina:** represents the player's energy and ability to perform actions. Stamina decreases with each attack, adding an extra layer of strategy to the game.

With the goal of boosting regular interactions between the server and it's members, I added a few more stats that players can improve over time by performing certain activies regularly. Conversely, players can lose stats by not interacting with the bot for a certain period of time.

Additionally I created some stats that are calculated based on the users fight history, so that past performance can affect future outcomes. 

* **Secondary Stats:**
  
  * **Morale:** A player's morale is affected by their recent fight history. Winning fights will boost morale, while losing fights will lower it. Morale affects a player's critical hit chance and overall performance in fights.
  * **Intimidation Level:** A player's intimidation level is based on their fight record, particularly the fighters history of finishing fights by KO/TKO. The higher the intimidation level, the more likely the opponent is to miss strikes or make mistakes during fights.

  * **Fighter Level:** I wanted to create a sense of progression in the game so that fight outcomes and daily interactions will improve the fighters and create interactions for users to assign bonus stats.    . Level advancement is drive how bonus stat points are granted.

  * **Bonuses:** Players can level up their fighter and choose bonuses that will affect their stats in fights. These bonuses can be selected when the player reaches the next fighter level and include: 
    * **Health Bonus:** Players can choose to boost their amount of health points. 
    * **Stamina Bonus:** Players can choose to boost their amount of stamina points.
    * **Damage Bonus:** Players can choose to boost their damage output in fights.

To encourage daily interactions, I created two bonus stats which would influence the fight calculations and outcomes: *Training level* and *Nutrition level*. These stats can be improved by performing daily training and diet activities, respectively. The higher the training level, the more effective the player's strikes will be in combat, while the higher the nutrition level, the faster the player's stamina will regenerate during fights.

I would need to create a background task that checks the last interaction time for each player and reduces their training and nutrition levels if they haven't interacted with the bot for a certain period of time. I took care of this in the main cog, bullshido.py, since I would be calling this function early from the main cog. 



# Let's Get Coding - Building the Main Components

## Player and Guild Configurations

Now that I had a high level plan for the cog structure and game mechanics, I started coding the core components of the cog. I began by setting up the player and guild configurations, which would store the game data and settings for each user and server. As I mentioned earlier, I used Red's Config object to manage these configurations. 

For the player configuration, I created a set of defaults with default_user = {}. The player configuration itself can include nested dictionaries, lists, and other data structures, which makes it easy to store complex player data. I added a nested dict where I could track not only wins, losses and draws but also the method for each win or loss, like UD (Unanimous Decision), SD (Split Decision), TKO (Technical Knock Out), KO (Knock Out). 

```python
#bullshido.py
class Bullshido(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.config = Config.get_conf(
            self, identifier=123123451514345671215451351235890, force_registration=True # This is a unique identifier for the cog configuration, and how Red determines which cog the configuration belongs to.
        )
        default_user = { # Default user configurations, including stats, levels, and other player data.
            "fighting_style": None,
            "wins": {"UD": 0, "SD": 0, "TKO": 0, "KO": 0},
            "losses": {"UD": 0, "SD": 0, "TKO": 0, "KO": 0},
            "draws": 0,
            "xp": 0,
            "level": 1,
            "level_points_to_distribute": 0,
            "stamina_bonus": 0,
            "health_bonus": 0,
            "damage_bonus": 0,
            "training_level": 1,
            "nutrition_level": 1,
            "morale": 100,
            "intimidation_level": 0,
            "stamina_level": 100,
            "health_points": 100,
            "prize_money_won": 0,
            "prize_money_lost": 0,
            "last_interaction": None,
            "last_command_used": None,
            "last_train": None,
            "last_diet": None,
            "fight_history": [],
            "permanent_injuries": [],
            "taunts": [],
        }
```

I ended up adding many configurations to the user config as I went along, suffering the usual amount of feature creep that comes with fun projects like this. Also growing in number of configuration items albeit to a lesser degree, was the guild configuration.

```python
default_guild = {
            "rounds": 3,
            "max_strikes_per_round": 5,
            "training_weight": 0.5, # Weight of training level in fight calculations, including strike damage and hit chance.
            "diet_weight": 0.3, # Weight of nutrition level in fight calculations, including stamina regeneration.
            "damage_bonus_weight": 0.5, # Weight of damage bonus in fight calculations, damage bonus is obtained with each level up.
            "base_health": 100, # Base health points for each player.
            "action_cost": 10, # Stamina cost for each action in a fight.
            "base_miss_probability": 0.15, # Base probability of missing a strike.
            "base_stamina_cost": 10, # Base stamina cost for each strike.
            "critical_chance": 0.1, # Base critical hit chance of 10%.
            "permanent_injury_chance": 0.5, # Chance that a critical hit will cause a permanent injury.
            "socialized_medicine": False, # Whether socialized medicine is enabled for the server.
            "socialized_medicine_payer_id": None, # User ID of the player who pays for socialized medicine.
        }
```

And lets register our configurations with Red's Config object, which will create the necessary data structures in the bot's database and allow us to read and write to them.

We will also create a background task to check for inactivity penalties every hour while the bot is ready. More on that in the next section.

We can initialize this task along with the other setup functions in the ```__init__``` method of the main cog file - bullshido.py. I also added a logger as always to help with debugging. 

```python


        self.config.register_user(**default_user)
        self.config.register_guild(**default_guild)
        self.setup_logging()
        self.bg_task = self.bot.loop.create_task(self.check_inactivity())
        self.logger.info("Bullshido cog loaded.")

```


### Player Inactivity Penalties

Since we already called check_inactivity in the cog __init__ we better create the method. 

I will check the last interaction time for each player and reduce their training and nutrition levels if they haven't interacted with the bot for a certain period of time.

```python
# bullshido.py
async def check_inactivity(self):
"""Check for inactivity penalites every hour while the bot is ready. """
        self.logger.info("Checking for inactivity...")
        await self.bot.wait_until_ready()
        while not self.bot.is_closed():
            await self.apply_inactivity_penalties()
            await asyncio.sleep(3600)  # Check every hour

    async def apply_inactivity_penalties(self):
        """ Define the inactivity penalties. """
        self.logger.info("Applying inactivity penalties...")
        current_time = datetime.datetime.utcnow()
        users = await self.config.all_users()
        for user_id, user_data in users.items():
            self.logger.info(f"Applying inactivity penalties for user {user_id}")
            await self.apply_penalty(
                user_id, user_data, current_time, "train", "training_level"
            )
            await self.apply_penalty(
                user_id, user_data, current_time, "diet", "nutrition_level"
            )

    async def apply_penalty(
        self, user_id, user_data, current_time, last_action_key, level_key
    ):
    """ Apply penalty for inactivity if the user did not interact with the bot for 2 days. """
        last_action = user_data.get(f"last_{last_action_key}")
        if last_action:
            last_action_time = datetime.strptime(last_action, "%Y-%m-%d %H:%M:%S")
            if current_time - last_action_time > timedelta(days=2):
                # Apply penalty if the user missed a day
                new_level = max(1, user_data[level_key] - 20)
                await self.config.user_from_id(user_id)[level_key].set(new_level)
                self.logger.info(
                    f"User {user_id} has lost 20 points in their {level_key.replace('_', ' ')} due to inactivity."
                )
                user = self.bot.get_user(user_id)
                if user:
                    await user.send(
                        f"You've lost 20 points in your {level_key.replace('_', ' ')} due to inactivity."
                    )
                await self.config.user_from_id(user_id)[f"last_{last_action_key}"].set(
                    current_time.strftime("%Y-%m-%d %H:%M:%S")
                )
```

Setting up our user configuration was essential for calculating inactivity penalties, since we needed a way to store data on the users last interaction to compare it to the current time.

### Fighting Styles and Strikes

![Alt text](/img/in-post/bullshido/Boxing-VS-Brazilain-Jiu-Jtist-1.png)


One of the key features of Bullshido is the ability to choose from a variety of fighting styles, each with its own unique set of strikes and abilities. I wanted to capture the essence of different martial arts disciplines, from the powerful strikes of Muay Thai to the precise movements of Karate. To achieve this, I created a dictionary of strikes for each fighting style, with associated damage ranges and again, spent way too long adding a range of actual strikes from each discipline.

```python
#fighting_constants.py
STRIKES = {
"Kung-Fu": {
        "Palm Strike": (5, 7),
        "Sweep": (4, 6),
        "Dragon Strike": (5, 7),
        "Tiger Palm": (5, 7),
        "Crane Kick": (6, 8),
        "Mantis Strike": (5, 7),
        "Iron Fist": (6, 8),
        "Phoenix Eye Fist": (5, 7),
        "Monkey Strike": (5, 7),
        "Dragon Fist": (6, 8),
        "Tiger Fist": (6, 8),
        "Crane Fist": (6, 8),
        "Mantis Fist": (5, 7),
        "Monkey Fist": (5, 7),
        "Panda Fist": (5, 7),
    }
"Karate": {
        "Straight Right": (5, 7),
        "Straight Left": (5, 7),
        "Left Cross": (5, 7),
        "Right Cross": (5, 7),
        "Hook": (5, 7),
        "Uppercut": (5, 7),
        "Backfist": (5, 7),
        "Ridgehand": (5, 7),
        "Knifehand": (5, 7),
        "Palm Strike": (5, 7),
        "Hammerfist": (5, 7),
        "Spearhand": (5, 7),
        "Reverse Punch": (6, 8),
        "Reverse Knifehand": (6, 8),
        "Reverse Hammerfist": (6, 8),
        "Reverse Spearhand": (6, 8),
        "Reverse Backfist": (6, 8),
        "Reverse Ridgehand": (6, 8),
        "Chop": (5, 7),
        "Knee Strike": (5, 7)
    }}
```
It was challenging coming up with a balanced number of strikes for each style, and I had to do a lot of research to make sure the strikes were accurate and representative of the fighting style. I also had to balance the damage ranges to ensure that no style was overpowered or underpowered compared to the others, which was a whole process which I will describe later. 

### Strike targeting and Injuries

Although the game is turn-based and doesn't involve direct player input, I wanted to add an element of excitement and realism by including realistic targeting and injury mechanics. Each strike in the game targets a specific body part, and depending on the strike's power and accuracy, it can cause different injuries to the opponent. I created a dictionary of body parts and associated injuries, which are randomly selected based on the strike's damage range and the opponent's health. 

I came up with a list of body parts. This was a fun exercise, and I tried to include a mix of common and less common body parts to add variety to the strikes.

```python
# fighting_constants.py
    BODY_PARTS = [
    "head", "chest", "left arm", "right arm", "nose", "neck", "left ear", "right ear",
    "teeth", "left leg", "right leg", "right foot", "left foot", "liver", "kidneys",
    "spine", "clavical", "left hip", "right hip", "left knee", "right knee", "solar plexus",
    "cranium", "kisser", "chin", "left shoulder", "right shoulder", "ass", "crotch", "spine", "ribs", "spinal column",
    "left ankle", "right ankle", "jaw", "left eye", "right eye", "stomach", "temple", "forehead", "mid-section","stomach",
    "throat", "ribcage", "butt"
]
```

That should do it for now. The next dictionary I created was for the purpose of mapping body parts to possible injuries. Carrying on with the over-the-top nature of martial arts movies, I wanted to include a variety of dramatic and sometimes humorous injuries that could occur during fights. This added an extra layer of excitement and unpredictability to the game. My aim is to make each fight unique and entertaining in some way.


```python
# fighting_constants.py
BODY_PART_INJURIES = {
    "head": [
        "Concussion", "Fractured Skull", "Cracked Skull", "Head Contusion", "Head Whiplash", "Brain Bleed", "Brain Swell",
        "Fractured Cranium", "Head Fracture", "Eye Socket Fracture", "Orbital Bone Fracture", "Cheekbone Fracture"
    ],
    "chest": [
        "Punctured Lung", "Shattered Rib", "Collapsed Lung", "Internal Bleeding", "Ruptured Spleen", "Cracked Rib",
        "Bruised Rib", "Bruised Kidney", "Lung Contusion", "Cracked Sternum", "Bruised Sternum", "Solar Plexus Contusion"
    ],
    "left arm": [
        "Dislocated Left Shoulder", "Fractured Left Arm", "Broken Left Arm", "Broken Left Collarbone", "Broken Left Wrist",
        "Broken Left Shoulder", "Sprained Wrist", "Deep Arm Cut", "Broken Elbow", "Torn Muscle", "Torn Ligament", "Severed Tendon", 
        "Torn Rotator Cuff", "Bicep Tear", "Tricep Tear", "Broken Finger", "Broken Hand", "Knuckle Fracture", "Broken Thumb", "Wrist Dislocation"
    ],
    "right arm": [
        "Dislocated Right Shoulder", "Fractured Right Arm", "Broken Right Arm", "Broken Right Collarbone", "Broken Right Wrist",
        "Broken Right Shoulder", "Sprained Wrist", "Deep Arm Cut", "Broken Elbow", "Torn Muscle", "Torn Ligament", "Severed Tendon", 
        "Torn Rotator Cuff", "Bicep Tear", "Tricep Tear", "Broken Finger", "Broken Hand", "Knuckle Fracture", "Broken Thumb", "Wrist Dislocation"
    ],
    "nose": [
        "Broken Nose", "Bloody Nose", "Cut Lip"
    ],
    "neck": [
        "Neck Strain", "Neck Fracture", "Whiplash", "Sprained Neck", "Cervical Fracture", "Spinal Disc Herniation"
    ],
    "left ear": [
        "Damaged Eardrum", "Ruptured Eardrum"
    ]}
    # And many many more injuries until I had a gruesome list of injuries for each body part!}
```

After I spent an entire day coming up with injuries and spending way too much time on learning the difference between abrasions and contusions, I remembered that AI would be perfect for this. I submitted my list of body parts and injuries so far to GPT-4 and after some prompting and a few iterations, I had a list of injuries that could rival a medical textbook.

### Strike and Grapple Verbs 

I wanted to add a bit of flair to the game by including a variety of strike and grapple verbs that would be used to concatenate with the strike names to create dynamic and exciting fight descriptions. I created a list of verbs for both striking and grappling, which would be randomly selected to describe the action of each strike or grapple during a fight. This really adds a layer of variety. 

I created `GRAPPLE_KEYWORDS` with a list of words that form part of my list of grapple moves from all styles. This is to ensure that we use a grapple action when the player attempts to initiate a grapple, or we could end up with a player "kicking" or "clobbering" a grapple move. 

`STRIKE_ACTIONS` is a list of verbs that will be used to describe the action of each strike during a fight,I tried thinking back to as many of those cartoon fight scenes as I could and came up with a list of verbs that should do for now. Personally, I would be satisfied with 'donks' being the only strike action but I'm sure the players will appreciate the variety. 

```python
# fighting_constants.py
GRAPPLE_ACTIONS = [
    "completes", "nails", "executes", "finishes", "performs", "grabs", "attains", "pulls off", "applies", "transitions to", "forces"]

GRAPPLE_KEYWORDS = [
    "Choke", "Mount", "Triangle", "Kimura", "Omoplata", "Crucifix", "Guillotine", "Lock", "Bar", "Hold", "Submission", "Crank", "Takedown", "Suplex", "Slam", "Throw", "Drag", "Take", "Redirect", "Chokehold", "Single Leg", "Double Leg", "Hug", "Toss", "Fireman", "Piledriver", "Full Nelson", "Cradle", "Clinch", "Tackle", "Backbreaker"
]

STRIKE_ACTIONS = [
    "throws", "slams", "nails", "whacks", "connects", "rips", "thuds", "crushes", "snaps", "smashes", 
    "pounds", "cracks", "hits", "drives", "lands", "bashes", "clobbers", "wallops", "hammers",
    "strikes", "belts", "mashes", "pummels", "slugs", "batters", "blasts", "bludgeons", "thumps", "thwacks",
    "wacks", "whams", "socks", "smacks", "beats", "whomps", "whips", "thwumps", "clangs", "claps", "donks"
]
```


### Adding Flavor to Fighting Events


```python
# fighting_constants.py
CRITICAL_MESSAGES = [
    "Channeling the power of the Dim Mak strike,",
    "Harnessing the technique bestowed upon them by the Black Dragon Fighting Society,",
    "Channeling the Grand Master Ashida Kim,",
    "Summoning the power of Count Dante,",
    "Channeling the power of the Black Dragon,",
    "Feeling the power of the Black Cobra,",
    "Channeling the Jin Rai of Ken Masters,",
    "Unleashing the power of E.Honda,",
    "Invoking the fire of Dhalsim,",
    "Sending a message from the spirit of Sagat,",
    "Unleashing the psycho power of M.Bison,",
    "Unfurling a Sonic Boom from the spirit of Guile,",
    "Spinning like Chun-Li,",
    "Dropping the pain like Zangief,",
    "With the swaying grace of Vega,",
    "Sending a message from the spirit of Ryu,",
    "Launching a Shoroyuken from the spirit of Ryu,",
    "Feeling the power of the Black Dragon,",
    "Harnessing the strength of Master Yoshi,",
    "Unleashing the Chi energy of The Red Tiger,",
    "Summoning the spirit of The White Crane,",
    "Calling upon the secrets of the Shadow Serpent,",
    "Harnessing the precision of Master Liu Kang,",
    "Channeling the ferocity of the Jade Dragon,",
    "Invoking the techniques of the Phoenix Warrior,",
    "Invoking the force of the Midnight Panther,"]
```


### Initialization

Let's import all of our fighting constants so we have access to all the damage ranges, body parts, injuries, strike actions, grapple actions and game flavor messages. 

```python
# fighting_game.py
from .fighting_constants import (
    STRIKES, CRITICAL_RESULTS, CRITICAL_MESSAGES, BODY_PARTS, GRAPPLE_KEYWORDS, GRAPPLE_ACTIONS, BODY_PART_INJURIES,
    STRIKE_ACTIONS, TKO_MESSAGES, KO_MESSAGES, KO_VICTOR_MESSAGE, TKO_VICTOR_MESSAGE, REFEREE_STOPS, FIGHT_RESULT_LONG,
    ROUND_RESULTS_WIN, ROUND_RESULTS_CLOSE, TKO_MESSAGE_FINALES, KO_VICTOR_FLAVOR)
```


When a match is initiated, the FightingGame class sets up the fight by initializing both fighters with their respective stats and passing them to a new instance of fighting_game.py.


## Next Time - The Fighting Game Flow and Presentation

Now we are ready to start building the game flow and mechanics. In the next section, we will dive into the fighting_game.py module and start coding the core game logic, including the fighting flow, strike calculations, presentation and more!

