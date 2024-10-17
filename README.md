<p align="center">
  <a href="https://hackathon.walter-group.com">
    <img alt="WALTER GROUP Hackathon: Sustainable Logistics" src="images/header.svg" width="500px" >
  </a>
  <h4 align="center">This is the central repository for participants of the <br><a href="https://hackathon.walter-group.com" target="_blank">WALTER GROUP Hackathon: Sustainable Logistics</a> which explains how to participate in the competition and links to all relevant resources like Replit agent templates.</h4>

  <p align="center">
    <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License"></img></a>
    <a href="https://www.walter-group.com"><img src="https://img.shields.io/badge/Organiser-WALTER%20GROUP-%2300529e" alt="Organiser WALTER GROUP"></img></a>
    <a href="https://join.slack.com/t/wg-hackathon-2024/shared_invite/zt-2ri6ti5h2-tmw7XY4CU7pdlx3jar6d0g"><img src="https://img.shields.io/badge/Slack-join%20chat-green" alt="Join Slack chat"></img></a>
  </p>
</p>

# Overview
Experts in data science, machine learning and software development will work together to find efficient and sustainable logistics solutions for the EU at the WALTER GROUP Hackathon: Sustainable Logistics on 17 October 2024.

Freight transports, delivery routes and logistics systems are key drivers of the European economy. They allow countries to work together across borders, achieve better results together and find efficient and sustainable solutions for the future through cooperation and collaboration. WALTER GROUP aims to foster this cooperation and brings together experts, developers and scientists to demonstrate what we can achieve by joining forces.

You do not have to be an expert in machine learning or data science. If you know how to code in one of our supported languages (C#, Java, Python) you are ready to go. A simple script making decisions via if / else branches might even be better than an involved machine learning model. We are excited to see what you can achieve!

# Simulation

We simulate a road network in Europe. For the purpose of the hackathon, it is simplified.

![image-20220512095546609](images/image-20220512095546609.png)

The world map for this road network is static and available: [map.json](data/map.json). **Map schema** is explained later.

The world map is represented by locations connected by a network of roads.  Some locations are special, as they produce goods, which will need shipping to other locations. 

Locations for producing and consuming goods are semi-deterministic and are not revealed publicly. You can analyse the **simulation trace data** to find out the patterns.

The participants of the competition will get to program trucks which can load the cargo and deliver it.  **The goal is to make money by delivering the cargo**. 

If you run out of money, your truck is suspended for the current simulation run.

This is very much like economic games you might have played in high school.

## Implementation

So how does the simulation work exactly? 

The simulation runtime is provided by WALTER GROUP and is responsible for keeping track of all stats like the current list of cargo offers (= produced goods), the trucks on the map, the current simulation date and time, fuel cost and many more. 

When the simulation is started it loads the world map, generates an initial list of cargo offers and spawns the trucks of the participants randomly on the map. Then the simulation will iterate through the list of trucks and will ask each one of them for their next move decision.

Afterwards, we simulate the passing of time via [discrete-event simulation](https://en.wikipedia.org/wiki/Discrete-event_simulation). Trucks travel, time passes and we jump forward to the next important event.

## Your Workflow

![image-20220513090946445](images/image-20220513090946445.png)

1. You code an agent that controls a truck.
2. The agent is developed and executed on Replit, a web-based IDE in the cloud.
3. Regularly (every 5 minutes) we start a new simulation run and send requests to all available agents.
4. You can observe the real-time dashboards during the run. 
5. You can also download the full simulation traces immediately after the run ends.

The goal is to **observe, iterate and improve**.

We have **a set of efficiency rules** at work:

- Your **truck gets money for delivering** cargo. Certain cargo types tend to have better margins out-of-the-box.
- Cargo delivery prices are market-driven. If a cargo isn't picked up, price for its delivery will increase over the time.
- You spend money on gas (fuel consumption uses COPERT formula for a loaded/empty truck and diesel price of `1.9l`)
- **Every week, you pay fixed operational costs**. So standing idle and doing nothing is a loosing strategy. You need to hustle to keep.
- If your balance is negative at the end of the week, the truck is bankrupt until the end of the simulation run.
- Weekly revenue is taxed progressively.
- **CO2 emissions** (computed via COPERT) have an additional associated cost (CO2 offset cost)

## Trucks

You control a truck by implementing a `decide` function. This function is called whenever the simulation doesn't have a plan for the truck (e.g. at the start of the simulation).

The simulation runtime will build a current world state for the truck (list of available cargo) and request a decision.

Trucks can decide to **deliver a cargo**, **drive to a specific location** empty, or **let the driver sleep** for a specific amount of time:

- `DELIVER CARGO_UID` (e.g. `DELIVER 23`) - The truck reserves a specific cargo offer, drives to the current cargo location, loads the cargo onto the truck, plans a route to the cargo destination, drives there and unloads the cargo. Fatigue and working hour effects can apply here, if _Sustainability Rules_ are in effect.
- `ROUTE LOCATION` (e.g. `ROUTE Berlin`) - A truck might decide to plan a route and drive to a specific location, although it is currently empty. For example a truck could anticipate, that a lucrative cargo offer will soon be available at a location and wants to minimize the pick up and delivery time.
- `SLEEP HOURS` (e.g. `SLEEP 1` or `SLEEP 8`) - When there are no cargo offers available, or when a truck driver is already on duty for a long time, it might be the best strategy to rest for a few hours. Note, that sleeping 8 times for one hour doesn't constitute a full rest.

Each action by a truck agent is an atomic action which will always succeed. After the decision is fully executed, the truck will be able to decide again.

## Simulation time
The current time is represented by the float `time` in the simulation. It represents the number of hours that have passed since the start of the simulation. 

- A value of 13.25 means `13:15`. 
- `26.0` means `1 day 2 hours`

Each action of a truck takes time. Driving a truck on the map, takes a certain amount of time, which is calculated by the distance and the speed limits of a route. The cargo offers delivery takes 5 hours, the truck will execute all necessary steps. Additional time might pass when _Sustainability Rules_ are active, and the driver gets into an accident due to insufficient resting time (see next section).

Example: 
- `loc`=A, `time`=8 
  
  The truck decides at 8:00 to drive from A to B, which will take 5.5 hours.

- `loc`=B, `time`=13.5

  The truck arrives at 13:30 at location B.

Hint: you can get time of the day by doing a modulo operation: `tod = time mod 24`

## CO2 emissions
Driving emits CO2 which is calculated via a simplified COPERT4 formula (see also https://www.eea.europa.eu/data-and-maps/data/external/copert-4). The simulation tracks the emissions for all trucks on the map. Try to keep your emissions as low as possible, while maximizing your profit.

CO2 emissions are counted as expensses, based on the CO2 offset cost per kg.

## Cargo offers

Simulation runtime tracks cargos available for delivery. Whenever it is time for a truck to make a decision, it will receive a personalized list of cargo offers. 

```json
{
  "uid": 100,                # unique cargo ID
  "origin": "Zaragoza",      # city to pick it up
  "dest": "Innsbruck",       # destination
  "name": "Fruits",          # kind
  "price": 1778.0,           # money you get for delivery
  "eta_to_cargo": 8.16,      # ETA from current loc to pickup loc
  "km_to_cargo": 780.0,      # kms to pickup location
  "eta_to_deliver": 27.76,   # ETA for picking up and delivering
  "km_to_deliver": 2730.0    # total kms to pick up AND deliver
}
```



ETA and KM estimates are relative to the current position of the truck. They account only for the driving time, but don't account for the loading/unloading and waiting times.

You can check out for the sample cargo offers in [data/](data/) directory.

# Available Data

We have a set of data available for the deeper analysis:

- World map (static)
- Simulation traces
- Observability Dashboard

## World Map

You can [download](data/map.json) the world map. It describes the road graph as a collection of locations and roads that connect them. Estimated driving speed is also provided.

## Simulation Traces

Simulation traces could be downloaded from the [traces folder](https://traces.endpoints.lkww-hackathon.cloud.goog). These are JSON files stored in [Chrome Tracing format](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/preview).

Note: Simulation traces that were published before the competition start - demonstrate the mechanics, but use a slightly different world. Keep that in mind, if you decide to analyse in advance.

You can analyse the traces manually or load into the `chrome://tracing` tool. 

Navigate to `chrome://tracing` in Chrome or Chromium browser, then drag-and-drop a trace file into the tracing UI.

There you can see a detailed breakdown of every single action that happened in the simulation, for all the participants and NPC agents.

![image-20220512105849865](images/image-20220512105849865.png)

You can zoom, filter and dig into the details. Events tend to have associated data:

![image-20220512110108793](images/image-20220512110108793.png)



### Caveat - ms instead of hours

Chrome tracing simulator isn't suitable for displaying large intervals like hours. So we "hack" and pass time and durations as milliseconds instead. When you see a millisecond unit, convert it mentally into hours.

E.g.:

- "Start 3,839.524 ms " means "Start at 3839.524 hours since the start of the simulation"
- "Wall duration 2.034 ms" means "Wall duration of 2.034 hours" (roughly two hours and a minute)



### Manual Analysis

The file looks like the snippet below. Actions of all teams are intertwined there:

```json
{"name": "DRIVE", "ph": "B", "ts": 18941.0, "pid": 5, "tid": 1, "args": {"a": "Salzburg", "b": "Linz", "fuel_l": 24.19, "co2_kg": 63.86, "cargo": 15, "km": 107, "delay": 0, "incident_cost": 0}},
{"name": "DRIVE", "ph": "E", "ts": 20156.90909090909, "pid": 5, "tid": 1},
{"pid": 5, "ts": 20156.90909090909, "ph": "C", "name": "balance", "args": {"value": 24806}},
{"name": "DECIDE", "ph": "i", "ts": 19000.0, "pid": 8, "tid": 0, "s": "t", "args": {"cmd": "SLEEP", "arg": "1"}},
{"name": "SLEEP", "ph": "B", "ts": 19000.0, "pid": 8, "tid": 0},
{"name": "SLEEP", "ph": "E", "ts": 20000.0, "pid": 8, "tid": 0},
{"pid": 8, "ts": 20000.0, "ph": "C", "name": "balance", "args": {"value": 8713}},
{"name": "DECIDE", "ph": "i", "ts": 19080.0, "pid": 4, "tid": 2, "s": "t", "args": {"cmd": "SLEEP", "arg": "1"}},
{"name": "SLEEP", "ph": "B", "ts": 19080.0, "pid": 4, "tid": 2},
{"name": "SLEEP", "ph": "E", "ts": 20080.0, "pid": 4, "tid": 2},
```

Unique identifiers are `pid` (team ID). You can look them up for your truck at the beginning of the file in the `M` (Metadata) records:

```json
{"name": "process_name", "ph": "M", "pid": 1, "args": {"name": "npc_truck_naive"}},
{"name": "thread_name", "ph": "M", "pid": 1, "tid": 0, "args": {"name": "npc_truck_naive/0"}},
{"name": "process_name", "ph": "M", "pid": 2, "args": {"name": "npc_truck_rinat"}},
{"name": "thread_name", "ph": "M", "pid": 2, "tid": 0, "args": {"name": "npc_truck_rinat/0"}},
{"name": "process_name", "ph": "M", "pid": 3, "args": {"name": "npc_truck_seb"}},
{"name": "thread_name", "ph": "M", "pid": 3, "tid": 0, "args": {"name": "npc_truck_seb/0"}},
{"name": "process_name", "ph": "M", "pid": 4, "args": {"name": "npc_fleet_profit_seeking"}}
```

If you have multple trucks, they will share the same `pid` but will have different `tid`.

## Observability Dashboard

We have  dashboards with real-time insight into the system across the simulation runs. 

Each run takes 5-10 minutes, and you can see it as a separate peak on the charts.

- [General dashboard](https://telemetry.endpoints.lkww-hackathon.cloud.goog/d/external/hackathon?orgId=1&refresh=30s)

![image-20220512110929530](images/image-20220512110929530.png) 

# Competition Results

Winners are calculated by running the simulation 3 times (to reduce the role of luck) at the end of the day and picking the winner based on the average final balance across all 3 simulation runs.

# Replit agent templates

**We recommend to perform the steps of this section at the very beginning of the Hackathon, just by using one of the supplied language templates and submitting your team's unique ID and Replit URL. After the agent is setup and you verified it is working as expected, you can iteratively improve the truck agents behavior.**

**Only one member of your team needs to follow these instructions!**

## 1. Create your own agent Repl from a template

To get you started quickly, we created several Replit templates for you. Depending on your language preference, click on one of the links below and then click the green `Use Template` button to create a new Repl. During that process you will likely have to sign up to Replit. No worries, a Replit account is for free and it includes a monthly quota of development time!

- [C# agent repository](https://replit.com/@sburgstaller/WALTER-GROUP-Hackathon-Sustainable-Logistics-C-Agent?v=1#README.md)
- [Java agent repository](https://replit.com/@sburgstaller/WALTER-GROUP-Hackathon-Sustainable-Logistics-Java-Agent?v=1#README.md)
- [Python agent repository](https://replit.com/@f-krause/WALTER-GROUP-Hackathon-Sustainable-Logistics-Python?v=1#README.md)

</br>
<img src="images/use-template.jpeg" width="640"/>
</br>

## 2. Start your agent and submit your team information

Click the green "> Run" button and wait. During the first start, the Repl will restore all its dependencies and will perform a full rebuild of the agent. When the agent is started, you will see a new tab in your Replit window called `Webview`. That means that your agent is up and running and will be able to service requests from the hackathon framework.

</br>
<img src="images/copy-url.jpeg" width="640"/>
</br>

Now we need to get the URL of your running Repl and submit it to the hackathon framework. Click on the Url of the Webview tab, or go to Networking in the left sidebar. Then click on the copy button for the Url.

Now open this form and enter your URL and your teams unique ID. Valid unique IDs consist of lower-case characters (a-z), numbers (0-9) and dashes (-). Dashes at the start, or end are not allowed.

**[IMPORTANT! SUBMIT team ID and Repl Url here](https://forms.gle/A6RFQg3qY1pnGAA7A)**

**If you do not submit your agent information via the above form, we will also not be able to send requests to your agent.**

## 3. Verify it worked

Within a few minutes your agent should be displayed on the hackathon dashboard and you should see request logs appearing in your Replit console. That is the sign that everything is working as expected and now you can start to improve your agent code.

# We are hiring

## The WALTER GROUP

- Management-run Austrian private group
- Established in 1924
- More than 4,800 employees
- Turnover for financial year 2021/2022: € 3.2 billion
- First class credit rating from international credit rating agencies

## Leading the way in Europe

The main business activities of WALTER GROUP include: **processing full truck loads** all over Europe via road and in combined transport through LKW WALTER, as well as **trading and leasing office, storage and sanitary containers** all over Europe through CONTAINEX, **investing in and leasing commercial and residential property** through WALTER BUSINESS-PARK, WALTER IMMOBILIEN and CONTAINEX IMMOBILIEN, **storage services** through WALTER LAGER-BETRIEBE and **leasing trailers and tractors** through WALTER LEASING.

## What you can expect here

Working in the WALTER GROUP means becoming part of **our big European family**. Because your future colleagues come from **41 countries** and speak **39 languages**. Nevertheless, we speak with one voice. Because several non-negotiable common interests unite us, which we are happy to share with you.

In the WALTER GROUP we see ourselves, first and foremost, as **facilitators** who **get things off the ground**. As such, we always deliver **top performance**, where others absolve themselves from responsibility saying "that's impossible".

As a European company, we are **used to push and exceed the limits**. This also applies to personal performance. Meaning, you always go the extra mile with us. Should you ever stumble, you can **rely on the help of your team**.

## Start your career here
We are hiring! If you are interested in working for WALTER GROUP, you can find all information about our employment opportunities [here](https://career.walter-group.com/uk/en/employment-opportunities#jobCategory=it).
