---
layout:     post
title:      "RaceFlow: Getting Faster with n8n"
subtitle:   "Using automation and AI to improve iRacing performance"
date:       2025-07-21 18:03:00
author:     "Dave C"
catalog: true
published: true
header-mask: 0.5
header-img: "img/in-post/raceflow/header.jpg"
tags:
  - simracing
  - n8n
  - automation
  - iracing
  - python
  - fastapi
---

# Introduction

I’m starting another one of those “this might be a terrible idea but let’s see where it goes” projects.

This one’s called **RaceFlow**.

It wasn’t some long-standing plan or grand vision. It actually came to me while I was first exploring **n8n**. 

<img src="/img/in-post/raceflow/n8n-logo.svg" alt="n8n logo" width="240rr" style="display:inline-block;vertical-align:middle;" />

After having just crawled out of the Home Assistant rabbit hole my friend **Pat** had sent me down this year, he suggested I try n8n. I was intrigued, and managed to deploy it in my homelab after hitting few snags. The solution involved deploying cloudflare funnels to expose n8n with DNS A record to a subdomain of a domain on a recognized registrar to allow for Google OAuth authentication *waffles on* *and on*.. but with some of his guidance I was able to get my server set up correctly.  

Honestly, after checking out n8n, the possibilities there are blowing my mind. I started out playing with some basic automations, like finally labelling my gmail inbox and clearing it of spam and promotions. After the flow was created which took some trial and error, mostly to grant n8n sufficient permissions to manage my inbox, the job was completed. In 4 hours n8n and ChatGPT 4o with a simple system prompt had completed something I had been putting off for several years. 

<img src="/img/in-post/raceflow/n8n_gmail.png" alt="My Gmail n8n flow" width="150%" />

It occurred to me how much automation will change with the inclusion of LLM's, and I started having a think about projects that would keep me building new flows.  While driving in my iracing simulator, I had an idea: 

> _“Wait, what if I could build an entire pipeline that combines iRacing results and telemetry, runs some AI analysis, and actually tells me how to improve?”_

Sounds awesome, but wait, there are services like **trophi.ai** and **Driver61** that already do a _really_ good job of this - mostly. They don't correlate telemetry data with real in-sim driver results. In an n8n flow I can combine analysis of telemetry data with actual results from the iracing data api - and there is potential for some real magic there. 

Besides, I’m not trying to replace them. I’m doing this _for the fun of it_—and to learn. I want to have local control of this process, data I can automate however I want and most importantly, the experience of building something like this end-to-end in n8n.

---
# What is RaceFlow supposed to be?

RaceFlow is my name for the n8n workflow I want to build, which 1. sounds cool and 2. will include these high level steps:

-  Fetch my race results, incidents, sessions, cars, tracks, and driver progression data from the iRacing data API. 
-  Gather telemetry files from our latest session or sessions from iRacing locally or manually defined telemetry files in ibt format. 
-  Parse telemetry (.ibt) files into usable JSON  
-  Break the data down corner by corner for a chosen track  
-  Compute time deltas and consistency metrics compared to benchmark
-  Apply basic coaching rules  
-  Make inference based on race results combined with telemetry
-  Presentation: Generate a report of the result. Text at first but ultimately a web based report.  

---

# Breaking Down the Flow Components 

Starting as I usually do with coding projects, I like to create a flowchart of how the application will work. I'm a visual learner, and while studying for my GPYC, physically drawing out the app control flow really helped me to understand Python conditionals. I grabbed my notebook and made a rough sketch of the nodes I might need, then transposed it into a mermaid flowchart. 

{% raw %}
```mermaid
flowchart TD
  subgraph "RaceFlow Pipeline"
    Z[" "] --> A["iRacing Data API"]
    A --> B["API Service Wrapper"]
    B --> C["n8n HTTP Request Node - Get Race / Lap / Driver Data"]
    C --> D["Telemetry File .ibt Uploaded to n8n"]
    D --> E["n8n Script Node: Telemetry Parser"]
    B --> F["Race Results JSON"]
    E --> G["Parsed Telemetry JSON"]
    F --> H["n8n Function Node: Corner Segmentation & Alignment"]
    G --> H
    H --> I["n8n Function Node: Delta Calculation"]
    I --> J["LLM or Rule-Based Node: Coaching Insights"]
    J --> K["n8n Report Generation Node HTML or PDF"]
    K --> L["n8n Notification"]

end
  ```
{% endraw %}


#### 1. iRacing Data API
The official iRacing backend where your race history, lap data, session metadata, and car/track info are stored.

---
#### 2. API Service Wrapper
A custom Python-based API layer that authenticates with iRacing and exposes key endpoints:
- Recent races
- Lap results
- Event logs
- Car/track info

This service acts as a local gateway, returning JSON-formatted data that n8n can consume.

---
#### 3. n8n HTTP Request Node (Get race & lap data) 
This node will call our API to get the data we need in json format. We will handle authentication in the API wrapper itself. 

---
#### 4. Telemetry File Upload (.ibt)
Telemetry files contain raw lap-by-lap data captured from the sim, and are usually available in the windows user profile c:\users\username\documents\iRacing\Telemetry. At this stage, create an n8n node to grab the latest file or accept a telemetry file supplied by the user. 

---
#### 5. n8n Script Node: Telemetry Parser  
I will use some open source Python tools to process the binary `.ibt` files into structured JSON containing:

- Lap times
- Sector splits
- Car inputs (Throttle, Brake, Gear Changes, Steering Angle)
- Car Conditions (RPM, Speed, Tire Wear, Tire Temperature, Engine Temperature, G Forces)
- Session Weather Conditions (Air Temperature, Track Temperature)
- Session Track Conditions (Rain, Marbles, Dirt)
- Session timing

It occurred to me that I'm assuming tools like this already exist and a cursory Github search confirmed several libraries will do the trick, including ```pyirt```. 

---
#### 6. Race Results + Parsed Telemetry (JSON)
Race metadata from iRacing and telemetry metrics are now available in parallel. Both datasets are essential to understanding performance **in context** (e.g., qualifying lap vs. race lap, weather, session type).

---
#### 7. n8n Function Node: Corner Segmentation & Alignment

- Merges both JSON sources and segments data by corner or section of the track.
- Matches telemetry laps with specific race laps
- Aligns sectors across sessions or stints
- Organizes data for delta calculation

---
#### 8. n8n Function Node: Delta Calculation 
This step calculates performance deltas (gains or losses) in each track segment. It factors in:

- Time gaps between laps
- Throttle/brake timing and intensity
- Brake and throttle application shape
- Steering consistency or overcorrection
- Whether you're losing time due to line, hesitation, or inputs

---
#### 9. LLM or Rule-Based Node: Coaching Insights 
Here's where things get cool. This node:

- Feeds the data into an LLM or runs rule-based logic
- Outputs natural-language coaching suggestions like:
        
	-  “You consistently lose time on entry to T3. Try braking 10m earlier.”
            
    - “Throttle modulation is smoother on your fastest laps. Reproduce that.”
            
    - “You're overdriving corner exits when under pressure.”
            
---
#### 10.  n8n Report Generation Node (HTML or PDF)
The insights are compiled into a readable, shareable format:

- HTML dashboard (At first .txt is fine)
- PDF report with sections and charts
- Optional data visualizations

---
#### 11. n8n Notification Node (Webhook) 

Finally, the finished report is delivered:
- Pushed to a Discord channel
- Stored to disk or Drive
- Triggered via discord webhook

---
### More Detailed Flow 

Having a better idea of the steps involved in the basic automation flow, I created a more detailed plan of how the automation might look in n8n. 

<img src="/img/in-post/raceflow/n8n_flow_raceflow.png" alt="Detailed Flow" width="150%" />

---
# Let's Get Started: iRacing Data API Wrapper 

## Why Build a Local API Wrapper?

Before n8n can automate anything, it needs a reliable, local source of iRacing data. The official iRacing Data API is powerful, but it’s not designed for easy, direct integration with tools like n8n. I wanted something that:

- Authenticates securely (no credentials in n8n flows)
- Returns clean JSON for any stat, result, or metadata I might want
- Runs locally, so I’m not rate-limited or dependent on a third-party service or source for iRacing data
- Is easy to extend as my needs grow

Enter: a FastAPI microservice.

<img src="/img/in-post/raceflow/fastapi.png" alt="Alt text" width="50%" />

---

### Step 1: Project Setup

I started with a minimal Python project:

```
iRacing-Data-API-Wrapper/

    ├── main.py

    ├── readme.md

    └── .env
```


- [main.py] will hold the FastAPI app and all endpoints.
- `.env` stores my iRacing credentials, not to be committed or can be set in the Docker run.
- [readme.md] documents the endpoints and usage.


### Dependencies


`pip install fastapi uvicorn iracingdataapi python-dotenv`

---

### Step 2: Authentication

The iRacing Data API requires an iRacing account to authenticate. This obviously has it's limitations, but since (for now) this automation is intended for my personal use only, supplying my credentials as environment variables will do. 

I used python-dotenv to load my iRacing username and password from the .env file. This keeps secrets out of my code and version control. If running as a container, I could supply these variables in the docker run. 

```python

from dotenv import load_dotenv
import os
load_dotenv()

USERNAME = os.getenv("IRACING_USERNAME")
PASSWORD = os.getenv("IRACING_PASSWORD")

if not USERNAME or not PASSWORD:    
    raise RuntimeError("IRACING_USERNAME and IRACING_PASSWORD must be set in .env")
```
---

### Step 3: Connecting to iRacing

The [iracingdataapi] package handles the heavy lifting of logging in and fetching data.

```python
from iracingdataapi.client import irDataClient

api = irDataClient(USERNAME, PASSWORD)
```

Now we have variable api which encapsulates the connection and credentials to the iRacing Data API, allowing us to make several subsequent requests to it for various kinds of data. 

---

### Step 4: Building the FastAPI App

FastAPI makes it easy to define endpoints. Here’s the basic app setup:

```python
from fastapi import FastAPI

app = FastAPI(

  title="iRacing Data FastAPI",

  description="API exposing iRacing data via iracingdataapi.client",

  version="1.0",)

```

---

### Step 5: Creating Endpoints

I wanted endpoints for all the data I might want to automate in n8n: member stats, recent races, lap data, cars, tracks, and more.

#### Example: Member Summary

```python
@app.get("/stats-member-summary")

async def stats_member_summary():

  results = api.stats_member_summary()

  if not results:

    raise HTTPException(status_code=404, detail="No member summary data found")

  return results

```


**Sample response:**

```json
{

 "cust_id": 123456,

 "this_year": {

  "num_league_sessions": 2,

  "num_league_wins": 1,

  "num_official_sessions": 20,

  "num_official_wins": 3

 }

}
```

---
#### Example: Recent Races

```python
@app.get("/stats-member-recent-races")

async def stats_member_recent_races():

  results = api.stats_member_recent_races()

  if not results or "races" not in results or not results["races"]:

    raise HTTPException(status_code=404, detail="No recent races found")

  return results
```

**Sample response:**

```json
{

 "cust_id": 123456,

 "races": [

  {

   "car_id": 169,

   "finish_position": 7,

   "incidents": 0,

   "laps": 15,

   "subsession_id": 73850366

  }

 ]

}
```

---
#### Example: Lap Data for a Subsession

Calling the `stats-member-recent-races` endpoint above gives a high level result of the session, but there is a lot more data that can be pulled here. The iRacing Data API splits practices, qualifying sessions or races into subsessions, with subsession_id being a key value associated with each of these events. Several of the API endpoints which will provide more detailed race data require subsession_id to be supplied with the API call, so gathering a list of recent subsessions is definitely useful here. 

Let's create a new endpoint, extracting the key values from the list of dictionaries returned by `stats-member-recent-races`.

```python
@app.get("/get-subsessions-recent-races")

async def get_subsessions_recent_races():

    races_data = api.stats_member_recent_races() 	# same as results in our previous method, but lets dig deeper

    races = races_data.get('races', []) # Lets take the list of races, being the key value of 'races' returned by the above method

    if not races:

        raise HTTPException(status_code=404, detail="No recent races found")

    subsession_ids = [race.get('subsession_id') for race in races if race.get('subsession_id') is not None] # From each race, let's get the subsession ID 

    if not subsession_ids:

        raise HTTPException(status_code=404, detail="No subsession IDs found in recent races")

    return {"subsession_ids": subsession_ids} # Return the subsession ids, which we can use later 
    return results
```

Now that we have subsession ID's for our recent events, we can get the real lap data for our driver and every other driver on the grid. We can use the `result_lap_data`  endpoint from the iRacing data API. 

```python
@app.get("/result-lap-data")

async def result_lap_data(subsession_id: int = Query(...)):

  results = api.result_lap_data(subsession_id)

  if not results:

    raise HTTPException(status_code=404, detail="No lap data found for the given subsession ID")

  return results
```


**Sample request:**  

`GET /result-lap-data?subsession_id=73850366`

**Sample response:**
```json

{

 "laps": [

  {

   "lap_number": 1,

   "lap_time": 92.123,

   "incidents": 0,

  }

 ]

}

``` 

---

### Step 6: Making It Discoverable

I wanted a friendly landing page at `/` that lists all endpoints, so I added a simple HTML response with all available routes and their descriptions. This makes it easy to see what’s available in the browser as a quick reference. 

FastAPI has a great feature allowing custom response classes, including `HTMLResponse` . By returning an [`HTMLResponse`](https://fastapi.tiangolo.com/advanced/custom-response/#html-response) from your endpoint, you can serve any HTML you like.  I made a basic page to serve up the list of endpoints: 

```python
@app.get("/", response_class=HTMLResponse) # 

async def index():

    endpoints = {

        "/": "iRacing Data FastAPI - Available API endpoints",

        "/stats-member-summary": "Get summary stats for the member",

        "/stats-yearly": "Get yearly stats for the member",

        "/stats-member-recent-races": "Get recent races for the member",

        "/get-subsessions-recent-races": "Get subsession IDs from recent races",

        "/get-results-recent-races": "Get results for recent races",

        "/get-result-events?subsession_id=<id>": "Get result events for a subsession",

        "/result-lap-data?subsession_id=<id>": "Get lap data for a subsession",

        "/stats-member-bests": "Get best stats for the member",

        "/stats_world_records?car_id=<car_id>&track_id=<track_id>": "Get world records for a car and track",

        "/get-tracks": "Get all tracks",

        "/season-list?year=<year>&quarter=<quarter>": "Get season list for a given year and quarter",

        "/season-race-guide": "Get the season race guide",

        "/series-stats": "Get series stats",

        "/member-profile": "Get member profile",

        "/cars": "Get car data",

        "/tracks": "Get track data",

        "/driver-list?page=<page>&per_page=<per_page>": "Get paginated driver list",

        "/member-chart-data": "Get member chart data",

        "/stats-member-career": "Get career stats for the member",

        "/series-seasons": "Get series seasons",

    }

    html = """

    <!DOCTYPE html>

    <html lang="en">

    <head>

        <meta charset="UTF-8">

        <title>iRacing Data FastAPI - API Endpoints</title>

        <meta name="viewport" content="width=device-width, initial-scale=1">

        <style>

            body {font-family: 'Segoe UI', Arial, sans-serif; background: #f8f9fa; margin: 0; padding: 0;}

            .container {max-width: 700px; margin: 40px auto; background: #fff; border-radius: 10px;

                        box-shadow: 0 2px 8px rgba(0,0,0,0.07); padding: 32px 40px 24px 40px;}

            h1 {color: #2d3748; font-size: 2rem; margin-bottom: 24px; text-align: center;}

            ul {list-style: none; padding: 0;}

            li {margin-bottom: 18px; padding: 16px; border-radius: 6px; background: #f1f3f6;

                transition: background 0.2s; display: flex; flex-direction: column;}

            li:hover {background: #e2e8f0;}

            .endpoint {font-family: 'Fira Mono', 'Consolas', monospace; color: #2563eb;

                       font-size: 1.05rem; margin-bottom: 4px; word-break: break-all;}

            .desc {color: #4b5563; font-size: 0.98rem;}

            @media (max-width: 600px) {.container {padding: 18px 8px;} h1 {font-size: 1.2rem;}}

        </style>

    </head>

    <body>

        <div class="container">

            <h1>iRacing Data FastAPI - Available API Endpoints</h1>

            <ul>

    """

    for endpoint, desc in endpoints.items():

        html += f"""

            <li>

                <span class="endpoint">{endpoint}</span>

                <span class="desc">{desc}</span>

            </li>

        """

    html += """

            </ul>

        </div>

    </body>

    </html>

    """

    return HTMLResponse(content=html)
```

---

### Step 7: Running the Service

To run the API locally:

```uvicorn main:app --reload```


Now, we can check [http://localhost:8000/](vscode-file://vscode-app/c:/Users/davec/AppData/Local/Programs/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-browser/workbench/workbench.html) for the endpoint list.

<img src="/img/in-post/raceflow/api_index.png" alt="Detailed Flow" width="120%" />

Bam! our iRacing data API wrapper is completed. I will just have to decide on the ideal place for it to run long term but I'm leaning toward containerization and managing it as a stack in portainer. 

---

### Step 8: Using the API in n8n

Now, n8n can call any of these endpoints using the HTTP Request node. For example, to get my latest race results:

- **Method:** GET
- **URL:** `http://localhost:8000/stats-member-recent-races`
- **Response:** JSON with all recent races

This means I can trigger automations, parse results, and combine them with telemetry data—all without ever exposing my iRacing credentials to n8n or the cloud.

## Next Up: Telemetry Parsing

With the API wrapper in place, the next step is to tackle telemetry file parsing and integration. Next time I’ll use Python and n8n to decode `.ibt` files and merge them with race results for deeper analysis!