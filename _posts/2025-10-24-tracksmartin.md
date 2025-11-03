---
layout:     post
title:      "TracksMartin: Hits on Demand"
subtitle:   "Building an Command-Line AI Music Production Tool with Click"
date:       2025-11-03 08:42:52
author:     "Dave C"
catalog: true
published: true
header-mask: 0.5
header-img: "img/in-post/tracksmartin/music-production-banner.jpg"
tags:
    - python
    - cli
    - click
    - generative-ai
    - music
    - suno
---

## Foreword

I'm a musician, and to be honest, I find the idea of AI making music kind of unsettling. It blurs the line between creativity and computation in a way that makes me very uneasy and worried about the future. But the cat's out of the bag - this technology exists, and ignoring it doesn't make it go away.

So I approached this as a technical hobbyist, trying to understand how it works and what it means, without pretending it's something it's not. The result was TracksMartin, a CLI tool that combines the Suno API for music generation with OpenAI's GPT-4o-mini for intelligent lyric writing. 

This isn't a guide to becoming an expert - it's a walkthrough from someone still learning, making mistakes, and figuring things out as they go. I learned a lot about Python's Click library in the process, and figured I'd document it for anyone else trying to build their first real command-line tool.

## Introduction

![TracksMartin](/img/in-post/tracksmartin/tracksmartin_purple.png)

The tool handles the entire workflow: from generating genre-appropriate lyrics to polling the Suno API for completion, and finally downloading

I've been working with Python for a few of years now, mostly building Discord bots and small automation scripts. My experience has been largely with discord.py and some basic API work, but I'd never built a proper command-line interface before. When I discovered the Suno API and started experimenting with AI-generated music, I realized I needed a better way to interact with it than writing one-off scripts every time I wanted to try something new.

I was also tired of having to enter in the detailed prompts and style tags to create a decent result. 

That's when I decided to learn Click, Python's command-line interface creation kit. I'd heard it was supposed to be the modern, easy-to-use option compared to argparse, and I figured building a music production tool would be a good way to force myself to actually learn it properly.

The project ended up being more ambitious than I initially planned - not just a simple API wrapper, but a full-featured tool with genre-aware lyric generation, interactive workflows, and automatic file management.

## Part 1: Planning and Structure

### Mapping Out the Project

Before writing any code, I sketched out what I actually wanted to build. This probably seems obvious, but I've wasted enough time diving into code only to realize halfway through that I structured everything wrong.

Here's what I knew I needed:

1. **A solid API wrapper** - Something that could handle all the Suno API endpoints cleanly
2. **Smart lyric generation** - Not just AI vomiting generic lyrics, but understanding genre conventions
3. **Both CLI and interactive modes** - Power users want commands; I wanted guidance
4. **Good error handling** - Because things always break

The folder structure I settled on:

```
tracksmartin/
├── tracksmartin.py           # CLI entry point (Click commands)
├── tracksmartin_api.py       # Suno API wrapper (the brains)
├── lyrics_generator.py       # OpenAI integration
├── genre_templates.py        # Genre-specific templates
├── requirements.txt
├── .env                      # API keys (never commit this!)
├── .gitignore
└── logs/                     # Daily log files
```

This separation made sense to me coming from Discord bot development - keep the interface, logic, and external communication separate. It's easier to test, easier to debug, and if I want to use the API wrapper in another project later, I don't have to drag Click along with it.

## Part 2: The API Wrapper - Getting Your Hands Dirty

### Starting Simple

I started with the basic class structure. Nothing fancy, just enough to make authenticated requests:

```python
# tracksmartin_api.py
import os
from typing import Optional, Dict, Any
from dotenv import load_dotenv
import requests

class SunoAPIError(Exception):
        """Custom exception for when things go wrong"""
        pass

class TracksMartinClient:
        """Client for talking to the Suno API"""
        
        def __init__(self, api_key: Optional[str] = None):
                load_dotenv()  # Load from .env file
                self.api_key = api_key or os.getenv("SUNO_API_KEY")
                
                if not self.api_key:
                        raise ValueError(
                                "API key required. Set SUNO_API_KEY in your .env file "
                                "or pass it directly."
                        )
                
                self.base_url = "https://api.sunoapi.com/api/v1"
                self.headers = {
                        "Authorization": f"Bearer {self.api_key}",
                        "Content-Type": "application/json"
                }
```

The pattern of "try environment variable first, fall back to parameter" is something I picked up from building Discord bots. It's convenient for development (just use .env) but flexible enough for other use cases. I added a .env_example to the repo so if you clone it, you know what to put in your .env.

### The Request Handler - Where Things Get Real

This is where I spent way more time than expected. Initial version was naïve - just make the request and hope it works. Reality hit fast.

```python
def _make_request(
        self, 
        method: str, 
        endpoint: str, 
        data: Optional[Dict] = None,
        params: Optional[Dict] = None
) -> Dict[str, Any]:
        """Make API requests with actual error handling"""
        url = f"{self.base_url}/{endpoint}"
        
        try:
                if method.upper() == "GET":
                        response = requests.get(url, headers=self.headers, params=params)
                elif method.upper() == "POST":
                        response = requests.post(url, headers=self.headers, json=data)
                else:
                        raise ValueError(f"Unsupported method: {method}")
                
                response.raise_for_status()
                return response.json()
                
        except requests.exceptions.RequestException as e:
                # This is the part that took me forever to get right
                error_msg = f"API request failed: {str(e)}"
                
                if hasattr(e, 'response') and e.response is not None:
                        try:
                                # Try to get JSON error details
                                error_detail = e.response.json()
                                error_msg += f"\nDetails: {error_detail}"
                        except:
                                # Fall back to raw text, but truncate it
                                error_msg += f"\nResponse: {e.response.text[:500]}"
                
                raise SunoAPIError(error_msg)
```

The key lesson here: **when API calls fail, you need context**. Just saying "request failed" is useless. The try/except dance of "attempt JSON parsing, fall back to text" saved me hours of debugging.

Also note the `[:500]` truncation - learned that one after a 10,000 character error dump filled my terminal.

### Creating Music - The Main Event

This is the workhorse method. It handles all the parameters the Suno API accepts, but makes most of them optional so users aren't overwhelmed:

```python
def create_music(
        self,
        prompt: str,
        title: Optional[str] = None,
        tags: Optional[str] = None,
        style_weight: Optional[float] = None,
        weirdness_constraint: Optional[float] = None,
        negative_tags: Optional[str] = None,
        custom_mode: bool = True,
        make_instrumental: bool = False,
        mv: str = "chirp-v5"
) -> Dict[str, Any]:
        """Create music with custom lyrics and style"""
        
        # Build payload incrementally
        payload = {
                "custom_mode": custom_mode,
                "make_instrumental": make_instrumental,
                "mv": mv
        }
        
        # Handle instrumental vs vocal
        if make_instrumental:
                payload["gpt_description_prompt"] = prompt
        else:
                payload["prompt"] = prompt
        
        # Only add optional params if provided
        if title:
                payload["title"] = title
        if tags:
                payload["tags"] = tags
        if style_weight is not None:  # Note: "is not None" for numbers!
                payload["style_weight"] = style_weight
        if weirdness_constraint is not None:
                payload["weirdness_constraint"] = weirdness_constraint
        if negative_tags:
                payload["negative_tags"] = negative_tags
        
        response = self._make_request("POST", "suno/create", data=payload)
        
        # Validate we got what we expected
        if "task_id" not in response:
                raise SunoAPIError(f"No task_id in response: {response}")
        
        return response
```

**Things I learned the hard way:**

1. **Check `is not None` for numeric values** - Using `if style_weight:` breaks when the value is `0.0`, which is valid but falsy
2. **Build payloads incrementally** - Cleaner than a giant dict with conditional entries
3. **Validate responses** - APIs lie. Check that you actually got a `task_id` before proceeding

### Polling - The Patience Game

Music generation takes time. The API returns a task_id immediately, then you poll until it's done. This was surprisingly tricky because the Suno API returns inconsistent response formats during generation:

```python
def poll_until_complete(
        self,
        task_id: str,
        max_attempts: int = 20,
        interval: int = 15,
        verbose: bool = True
) -> Dict[str, Any]:
        """Poll until complete or we give up trying"""
        
        for attempt in range(1, max_attempts + 1):
                time.sleep(interval)
                response = self.get_music(task_id)
                
                # Handle all the different response states
                if response.get('type') == 'not_ready':
                        if verbose:
                                print(f"Attempt {attempt}/{max_attempts}: Not ready...")
                        continue
                
                if response.get('code') != 200 or 'data' not in response:
                        if verbose:
                                print(f"Attempt {attempt}/{max_attempts}: Waiting for data...")
                        continue
                
                clips = response['data']
                
                if not clips:
                        if verbose:
                                print(f"Attempt {attempt}/{max_attempts}: No clips yet...")
                        continue
                
                clip = clips[0]
                state = clip.get('state')
                
                if state == "succeeded":
                        if verbose:
                                print(f"\n✓ Complete! (Task: {task_id})")
                        return clip
                
                elif state in ["error", "failed"]:
                        raise SunoAPIError(f"Generation failed: {state}")
                
                if verbose:
                        print(f"Attempt {attempt}/{max_attempts}: {state}...")
        
        # Ran out of attempts
        raise SunoAPIError(
                f"Timeout after {max_attempts * interval} seconds"
        )
```

The multiple `continue` statements handle all the weird edge cases I encountered. Sometimes `response.get('type')` is "not_ready", sometimes there's no data yet, sometimes data is an empty list. Handle all the cases or suffer mysterious failures.

The `verbose` flag is crucial - waiting 5 minutes with no feedback feels like the program froze.

## Part 3: Click - Making It User-Friendly

### Why Click?

Coming from Discord bot development, Click's decorator syntax felt immediately familiar. Compare this to argparse:

**Click:**
```python
import click

@click.command()
@click.option('--name', help='Your name')
def greet(name):
        click.echo(f"Hello, {name}!")
```

**argparse:**
```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument('--name', help='Your name')
args = parser.parse_args()
print(f"Hello, {args.name}!")
```

Click feels cleaner to me. Decorators are self-documenting, and you don't have that awkward `parser.parse_args()` dance. I'd actually tried to build CLI's with argparse before and it just never made sense to me, and I always ended up frustrated.

### Building the Command Group

Everything starts with a command group:

```python
@click.group(context_settings=dict(help_option_names=['-h', '--help']))
@click.version_option(version='2.0.0', prog_name='TracksMartin CLI')
def cli():
        """TracksMartin - Hits on Demand.
        
        AI-powered music production from the command line.
        """
        pass  # Groups need a function body, even if it does nothing
```

The `pass` statement confused me at first - why have an empty function? Because Click needs something to attach the decorators to. The actual work happens in the subcommands.

### The Create Command - Learning by Doing

This command grew organically as I figured out what options were actually useful. Started simple, added complexity as needed:

```python
@cli.command()
@click.option('--title', '-t', help='Song title')
@click.option('--prompt', '-p', help='Lyrics or description')
@click.option('--prompt-file', '-f', type=click.File('r'), 
                            help='Read lyrics from file')
@click.option('--tags', help='Style tags (genre, mood, tempo, etc.)')
@click.option('--style-weight', type=float,
                            help='Style adherence (0.0-1.0)')
@click.option('--weirdness', type=float,
                            help='Creativity level (0.0-1.0)')
@click.option('--instrumental', is_flag=True,
                            help='No vocals, just instruments')
@click.option('--model', '-m',
                            type=click.Choice(['chirp-v3-5', 'chirp-v4', 'chirp-v5']),
                            default='chirp-v5', show_default=True)
@click.option('--wait/--no-wait', default=True,
                            help='Wait for generation')
@click.option('--download/--no-download', default=True,
                            help='Download when complete')
@click.option('--output-dir', type=click.Path(exists=True),
                            default='.', help='Where to save files')
def create(title, prompt, prompt_file, tags, style_weight, weirdness,
                     instrumental, model, wait, download, output_dir):
        """Create a new song"""
        
        # Handle file input
        if prompt_file:
                prompt = prompt_file.read()
        
        # Validate inputs
        if not prompt:
                click.secho("Error: Need --prompt or --prompt-file", 
                                     fg='red', err=True)
                sys.exit(1)
        
        if not title:
                click.secho("Error: Need --title", fg='red', err=True)
                sys.exit(1)
        
        # Do the thing
        client = TracksMartinClient()
        
        try:
                click.secho(f"\nCreating: {title}", fg='cyan', bold=True)
                
                response = client.create_music(
                        prompt=prompt,
                        title=title,
                        tags=tags,
                        style_weight=style_weight,
                        weirdness_constraint=weirdness,
                        make_instrumental=instrumental,
                        mv=model
                )
                
                task_id = response['task_id']
                click.secho(f"✓ Task created: {task_id}", fg='green')
                
                if wait:
                        clip = client.poll_until_complete(task_id, verbose=True)
                        
                        if download and clip.get('audio_url'):
                                filename = client.sanitize_filename(title) + ".mp3"
                                filepath = f"{output_dir}/{filename}"
                                
                                with click.progressbar(length=1, label='Downloading') as bar:
                                        client.download_file(clip['audio_url'], filepath)
                                        bar.update(1)
                                
                                click.secho(f"✓ Saved: {filepath}", fg='green')
        
        except Exception as e:
                click.secho(f"✗ Error: {e}", fg='red', err=True)
                sys.exit(1)
```

**Click patterns I learned:**

- `type=click.File('r')` - Automatic file handling, no manual open/close
- `is_flag=True` - Boolean flags that don't need values
- `type=click.Choice()` - Validation with autocomplete
- `--option/--no-option` - Boolean toggles with defaults
- `click.secho()` - Colored output (makes errors obvious)
- `click.progressbar()` - Simple progress indication

The colored output is more important than I initially thought. When you're scrolling through terminal output, having errors in red and success in green helps *so much*.

## Part 4: Genre-Aware Lyrics

### The Problem with Generic Prompts

My first attempt at lyric generation was embarrassingly simple:

```python
# Don't do this
result = openai.chat.completions.create(
        messages=[{"role": "user", "content": "Write a song about love"}]
)
```

The result? Generic slop that could be any genre. Turns out, a country song about love is fundamentally different from a hip-hop song about love - not just in sound, but in structure, language, and conventions.

### Building the Genre Templates

I spent time actually analyzing songs in different genres. What makes country *country*? What makes hip-hop *hip-hop*? Here's what I learned:

```python
# genre_templates.py
GENRE_TEMPLATES = {
        "pop": {
                "structure": "Verse 1, Pre-Chorus, Chorus, Verse 2, Pre-Chorus, Chorus, Bridge, Chorus",
                "characteristics": [
                        "Catchy, repetitive hooks",
                        "Simple vocabulary, relatable themes",
                        "Radio-friendly 3-4 minute length",
                        "Emphasis on memorable chorus"
                ],
                "style_notes": "Pop is about immediate emotional connection. Keep it simple, make it memorable."
        },
        "hip-hop": {
                "structure": "Intro, Verse 1, Hook, Verse 2, Hook, Verse 3, Hook, Outro",
                "characteristics": [
                        "Complex wordplay and metaphors",
                        "Strong rhythmic flow",
                        "Internal rhyme schemes",
                        "Authentic storytelling or confident braggadocio"
                ],
                "style_notes": "Hip-hop demands technical skill. Every bar should have purpose. Flow matters."
        },
        # ... more genres
}
```

This might seem like overkill, but it dramatically improved the results. The templates give GPT-4o-mini concrete guidance instead of vague instructions.

### The Lyrics Generator

```python
# lyrics_generator.py
from openai import OpenAI
from genre_templates import GENRE_TEMPLATES

class LyricsGenerator:
        def __init__(self, api_key: Optional[str] = None):
                load_dotenv()
                self.api_key = api_key or os.getenv("OPENAI_KEY")
                self.client = OpenAI(api_key=self.api_key)
        
        def generate_lyrics(
                self,
                theme: str,
                genre: str = "pop",
                title: Optional[str] = None,
                mood: Optional[str] = None,
                length: str = "medium"
        ) -> Dict[str, str]:
                """Generate genre-appropriate lyrics"""
                
                genre_info = self.GENRE_TEMPLATES.get(genre.lower(), {})
                
                # Build detailed system prompt
                system_prompt = f"""You are an expert {genre} songwriter.

GENRE CHARACTERISTICS:
{chr(10).join('- ' + c for c in genre_info['characteristics'])}

TYPICAL STRUCTURE:
{genre_info['structure']}

STYLE NOTES:
{genre_info['style_notes']}

Write authentic {genre} lyrics using proper structure tags: [Verse 1], [Chorus], etc."""

                # Build user prompt with specifics
                user_prompt = f"""Write {genre} lyrics about: {theme}

Length: {self._get_length_description(length)}
Genre: {genre}"""
                
                if title:
                        user_prompt += f"\nTitle: {title}"
                if mood:
                        user_prompt += f"\nMood: {mood}"
                
                user_prompt += """

Format your response as:
TITLE: [song title]
TAGS: [3-5 style tags for Suno]

[Intro]
(if needed)

[Verse 1]
(lyrics)

[Chorus]
(lyrics)

Make it authentic {genre}, not generic."""

                response = self.client.chat.completions.create(
                        model="gpt-4o-mini",
                        messages=[
                                {"role": "system", "content": system_prompt},
                                {"role": "user", "content": user_prompt}
                        ],
                        temperature=0.8  # Higher for creativity
                )
                
                # Parse the response
                content = response.choices[0].message.content
                
                # Extract title, tags, and lyrics
                # (parsing logic here)
                
                return {
                        'lyrics': lyrics,
                        'title': title,
                        'suggested_tags': tags
                }
```

**Key insights:**

1. **Separate system and user prompts** - System defines the role/expertise, user defines the task
2. **Concrete examples in prompts** - "Use [Verse 1], [Chorus]" beats "use structure tags"
3. **Temperature matters** - 0.8 adds creativity while keeping coherence
4. **Parse carefully** - GPT-4o-mini doesn't always follow format perfectly

### Integrating with the CLI

Adding auto-lyrics to the create command:

```python
@cli.command()
@click.option('--auto-lyrics', is_flag=True,
                            help='Generate lyrics with AI')
@click.option('--theme', help='Theme for lyrics (required with --auto-lyrics)')
@click.option('--genre', '-g', help='Musical genre')
@click.option('--mood', help='Mood/tone')
# ... other options ...
def create(auto_lyrics, theme, genre, mood, ...):
        """Create a song"""
        
        if auto_lyrics:
                if not theme or not genre:
                        click.secho("Error: Need --theme and --genre with --auto-lyrics",
                                             fg='red')
                        sys.exit(1)
                
                click.secho("\n🎵 Generating lyrics...", fg='cyan')
                
                lyrics_gen = LyricsGenerator()
                result = lyrics_gen.generate_lyrics(
                        theme=theme,
                        genre=genre,
                        mood=mood,
                        title=title
                )
                
                prompt = result['lyrics']
                title = result['title']
                tags = result['suggested_tags']
                
                click.echo("\n" + "="*60)
                click.echo(prompt)
                click.echo("="*60)
        
        # Continue with music creation...
```

Now you can go from concept to complete song in one command:

```bash
tracksmartin create \
    --auto-lyrics \
    --theme "summer romance" \
    --genre "pop" \
    --mood "upbeat"
```

## Part 5: Interactive Mode - Making It Beginner-Friendly

### Why Interactive Mode?

I'm comfortable with command-line tools. But not everyone is. Interactive mode bridges that gap by guiding users through the process step by step.

```python
@cli.command()
def interactive():
        """Interactive creation session"""
        
        click.secho("\nTracksMartin - Interactive Mode", fg='cyan', bold=True)
        
        try:
                # Ask questions progressively
                auto_gen = click.confirm("Auto-generate lyrics?", default=False)
                
                if auto_gen:
                        # Show genre list
                        genres = LyricsGenerator.list_supported_genres()
                        click.echo("\nGenres:")
                        for i, g in enumerate(genres, 1):
                                click.echo(f"  {i}. {g}")
                        
                        # Let them pick by number or name
                        genre_input = click.prompt("Genre (name or number)", type=str)
                        
                        try:
                                num = int(genre_input)
                                genre = genres[num - 1] if 1 <= num <= len(genres) else genre_input
                        except ValueError:
                                genre = genre_input
                        
                        theme = click.prompt("Theme/topic")
                        mood = click.prompt("Mood", default="")
                        length = click.prompt("Length",
                                                                 type=click.Choice(['short', 'medium', 'long']),
                                                                 default='medium')
                        
                        # Generate and show
                        lyrics_gen = LyricsGenerator()
                        result = lyrics_gen.generate_lyrics(...)
                        
                        click.echo(result['lyrics'])
                        
                        if not click.confirm("Use these lyrics?", default=True):
                                return
                
                else:
                        # Manual entry path
                        title = click.prompt("Title")
                        click.echo("Enter lyrics (Ctrl+D when done):")
                        
                        lyrics_lines = []
                        try:
                                while True:
                                        lyrics_lines.append(input())
                        except EOFError:
                                pass
                        
                        prompt = '\n'.join(lyrics_lines)
                
                # Configure style
                tags = click.prompt("Style tags")
                
                # Create music
                client = TracksMartinClient()
                response = client.create_music(...)
                
                if click.confirm("Wait for completion?", default=True):
                        clip = client.poll_until_complete(...)
                        
                        if click.confirm("Download?", default=True):
                                # Download logic
                                pass
        
        except KeyboardInterrupt:
                click.echo("\nExiting...")
```

**Interactive patterns:**

- `click.confirm()` - Yes/no questions
- `click.prompt()` - Text input with optional defaults
- `type=click.Choice()` - Selection from options
- Handle `KeyboardInterrupt` gracefully for Ctrl+C

The multi-line input is a bit janky (Ctrl+D on Unix, Ctrl+Z on Windows) but it works well enough for lyrics.

## Part 6: What I'd Do Differently

### Testing - My Biggest Regret

I built this entire thing without tests. Then I refactored the API wrapper and broke three commands. Here's what I should have done from day one:

```python
# test_tracksmartin_api.py
import pytest
from tracksmartin_api import TracksMartinClient

def test_sanitize_filename():
        """Basic utility testing"""
        assert TracksMartinClient.sanitize_filename("Test Song!") == "Test Song"
        assert TracksMartinClient.sanitize_filename("../evil") == "evil"

def test_client_requires_key():
        """Validate initialization"""
        with pytest.raises(ValueError):
                TracksMartinClient(api_key=None)

@pytest.mark.integration
def test_create_music():
        """Integration test (hits real API)"""
        client = TracksMartinClient()
        response = client.create_music(
                prompt="Test",
                title="Test Song",
                make_instrumental=True
        )
        assert 'task_id' in response
```

Testing isn't exciting, but it saves so much pain later. Start small - test the utilities first, then the core logic, then integration tests.

### Logging - Added Too Late

I generated dozens of songs before realizing I had no record of what I'd made. Added logging retroactively:

```python
# tracksmartin.py
import logging
from datetime import datetime

def setup_logging():
        log_dir = "logs"
        os.makedirs(log_dir, exist_ok=True)
        
        log_file = os.path.join(
                log_dir,
                f"tracksmartin_{datetime.now().strftime('%Y%m%d')}.log"
        )
        
        logging.basicConfig(
                level=logging.INFO,
                format='%(asctime)s - %(levelname)s - %(message)s',
                handlers=[
                        logging.FileHandler(log_file),
                        logging.StreamHandler()
                ]
        )
        
        return logging.getLogger(__name__)

logger = setup_logging()

# Then use it everywhere
logger.info(f"Creating: {title}")
logger.info(f"Task ID: {task_id}")
logger.info(f"Downloaded: {filepath}")
```

Daily log files keep things manageable. After a month, you can delete old logs without losing recent history.

### Configuration Files

Typing the same style preferences repeatedly gets old. Should have added a config file:

```yaml
# ~/.tracksmartin/config.yaml
defaults:
    model: chirp-v5
    style_weight: 0.7
    output_dir: ~/Music/TracksMartin
    tags: "pop, upbeat, 120 bpm"
```

Then merge config values with CLI arguments (CLI takes precedence).

## Conclusion

Building TracksMartin taught me that CLI tools are fundamentally different from web apps or Discord bots. You're not just writing code that works - you're designing an interface someone has to *type at*. Every option name matters. Every error message matters. Every default value matters.

**What I learned:**

- Click's decorator system makes CLI development feel natural
- Good error messages require more thought than I expected
- Genre-specific templates dramatically improve AI output
- Interactive mode serves a different audience than CLI commands
- Logging should be there from day one
- Testing should be there from day one (seriously, test your code)

**What worked:**

- Separating API wrapper, lyrics generator, and CLI into distinct modules
- Building both command-line and interactive interfaces
- Using genre templates instead of generic prompts
- Comprehensive logging for debugging and records

**What didn't:**

- Skipping tests initially
- Not planning for configuration files
- Underestimating how long polling logic would take

The complete project is on [GitHub](https://github.com/ropeadope62/tracksmartin). Feel free to use it, fork it, or just poke around to see how it works.

Just remember - true artistry comes from human experience, emotion, and connection. This is just a tool to explore the technology, not a replacement for actual creativity.

## Resources That Helped

- [Click Documentation](https://click.palletsprojects.com/) - Comprehensive and well-written
- [Requests Documentation](https://requests.readthedocs.io/) - HTTP for humans
- [OpenAI Cookbook](https://cookbook.openai.com/) - Prompt engineering patterns
- Stack Overflow - For all the weird edge cases

## Quick Reference

**Essential Click patterns:**

```python
# Basic command
@cli.command()
def simple():
        click.echo("Hello")

# With options
@cli.command()
@click.option('--name', required=True)
def greet(name):
        click.echo(f"Hi, {name}")

# File input
@cli.command()
@click.option('--file', type=click.File('r'))
def read(file):
        content = file.read()

# Boolean flag
@cli.command()
@click.option('--verbose', is_flag=True)
def run(verbose):
        if verbose:
                click.echo("Verbose mode")

# Progress bar
@cli.command()
def download():
        with click.progressbar(range(100)) as bar:
                for i in bar:
                        time.sleep(0.01)
```

**Common pitfalls:**

- Using `if value:` instead of `if value is not None:` for numeric checks
- Not handling all API response formats during polling
- Forgetting to validate response structure before accessing nested data
- Not truncating error messages (hello 10KB dumps)
- Skipping tests because "it's just a small project"

*This project was built while learning. Mistakes were made. Lessons were learned. Code works now.*


