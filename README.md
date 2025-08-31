# Trip Planner MCP 
## Introduction to MCP
This tutorial walks you through configuring MCP servers in VS Code using the [Cline](https://docs.cline.bot/getting-started/installing-cline) extension.
You’ll learn how to install the tools, set up Node.js, and configure servers like Firecrawl and Baidu Map.
Most importantly, you will see the mechanism behind all the processes!

## What you will learn
- What the MCP client is and how it connects LLMs with MCP servers with Cline
- How to install and set up the environment
- How to configure MCP servers with API keys
- Mechanism of MCP with a real use case

## Prerequisites
- VS Code installed
- API keys for the MCP servers you want to use.

## Let's go!
Before we start, I would like to present the three key components:
  1. **LLM**: The “brain” of the system. It understands your natural language request and decides when to call a tool to get real data.
  3. **MCP Client**: This is the “connector” or port. Its job is to let the LLM talk to MCP servers and to forward requests/responses between them.
  4. **MCP Server**: The tools that really do the job, such as web crawling. It describes to the client what functions it provides, waits for the LLM’s request, and then sends back results.

### Step 1: Cline Installation 
The first step is to install an MCP client. Cline is one of the easiest MCP clients to use as it is integrated into VS Code. You can just search and download Cline in the extensions marketplace of VS Code:

![Cline](./images/cline.png)

### Step 2: Choose your favourite LLM
The second step is to configure your LLM. You can choose your preferred LLM from multiple sources (OpenAI, Gemini, Claude, etc.) 
The fastest way is to get an [OpenRouter](https://openrouter.ai/) API key as they provide free LLMs from families as Deepseek, Qwen, and Mistral, etc. 

<img src="./images/LLM_configuration.png" alt="LLM" width="40%"/>



### Step 3: Set up your CMCP server.
First of all, you need to download [Node.js].(http://Node.js) Many MCP servers require Node.js to run because most of them are written in JavaScript/TypeScript, and Node.js provides the runtime environment needed to execute these programs.

Then, create a configuration file in the root directory to register our MCP servers. This file instructs the MCP client (Cline) on which servers to load, how to start them, and what credentials are required. ** Don't worry, you can find how to fill it in the official document of each MCP server, and you need to get an api key for each of them.**

In our case, we configured two popular MCP servers: Firecrawl for web crawling, and Gaode Map for location search and transport planning. 

```json
{
  "mcpServers": {
    "firecrawl-mcp": {
      "timeout": 60,
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "firecrawl-mcp"
      ],
      "env": {
        "FIRECRAWL_API_KEY": "your api key"
      }
    },
    "amap-maps": {
      "timeout": 60,
      "command": "npx",
      "args": [
        "-y",
        "@amap/amap-maps-mcp-server"
      ],
      "env": {
        "AMAP_MAPS_API_KEY": "your api key"
      },
      "type": "stdio"
    }
  }
}
```

Once you fill this JSON file, you can check in Cline if they are correctly configured:

<img src="./images/check_server.png" alt="MCP server check" width="30%"/>

Now the configuration is done, we can start a chat in Cline to try it out! 

## Real Use Case: Planning a One-Day Trip in Haikou City

In this example, we will combine two MCP servers to plan a one-day trip in my hometown, Haikou City, in the lovely tropical island of China. 

Remember we have two MCP servers: 

- **Firecrawl**  
  - Takes a web URL as input and returns website content.  
  - Its coolest feature is that you can define a **JSON schema**, and it will intelligently extract structured information from the web page according to that schema.  
  - This saves a lot of time compared to manually processing raw text.

- **Gaode Map (amap-maps)**  
  A map MCP powered by one of the most popular map applications in China.  
  It provides tools such as:
  - Converting place names into geographic coordinates  
  - Weather search  
  - Route planning (driving, walking, bicycling, public transit)  
  - Distance calculation

### Use Case Description

Our scenario is as follows:

1. Extract **3 attractions** from a travel blog, including their **name, location, and description**.  
2. Send these places to the **Map MCP** to generate a **public transportation route**.  
3. Combine the extracted information and the route plan into a **one-day travel itinerary** with all details:
   - Which places to visit  
   - What activities to do at each place  
   - How to get from one place to another using public transportation

All we need to do is to give a prompt, the LLM will select the best MCP server and its tool to do all the job for us! 

<pre> ```
Background:
I am traveling to Haikou city in China, and I am especially interested in cultural and historical attractions.

Task:
Extract 3 tourist attractions from a travel website:https://www.chinadiscovery.com/hainan/haikou/things-to-do.html 
That aligns with my interest, then use the maps MCP to generate a travel plan.

Steps:
1. Use the web crawling MCP firecrawl_extract to get attractions from the website in JSON format. 
   Output JSON schema: 
   - Places to Visit: name of the place 
   - location: Chinese address or landmark, required in Chinese for map accuracy
   - Description: a description of the place and things to do in English
   - Recommend Length of Visiting: time for visit 

Note: Prefer cultural and historical attractions if possible. Then you select only three of them that aligns most to my interest. 

2. Use maps_geo to convert each location into coordinates.

3. Plan the trip: start from the first attraction, then go to the second, then the third. 
   Use maps_direction_transit_integrated to get routes with public transportation routes. I have no car. 

4. Generate a final travel plan in Markdown text:
   - Attraction name + things_to_do
   - Route details between attractions with public transportation
   - Full, readable itinerary with cultural highlights
  - You should organize my day with a plan to make sure I have a good time in Hiakou City
``` </pre>

### Model Response
The LLM immediately decides to call the Firecrawl MCP server and selects the tool "firecrawl_extract
" from its toolbox. 
(image)
This is because, as shown here, in the tool box of each MCP server, each tool has a description of what it is capable for and what kind of paramter it needs to perform the job. The LLM analyse the human request first and look through all the availiable tools to select the one whose description aligns the most to the request. 

Then it generated a request for Firecrawl in JSON format, filling all the parameters needed for the tool. 

Here is exactly the essence of MCP: translating a human request to json request that applicable for all the MCP servers. 
That´s why in the definition we say, MCP is the port that connects LLM and external tools to aquire information. 

The json request will be sent to the selected MCP server once the user confirm the request. Only in a few seconds, we recieved main torist spots in Haikou, each spot has the information we need, just as defined in our schema. 

Then our brain, the LLM decides to call the next MCP server: Ama map. It decides to search the places to get the coordinates of them. So it first uses the maps_geo tool. After obtaining all the geografic information of the three places, it needs to plan the route by calling a new tool: maps_direction_transit_integrated, which can give public transportation routes when it has coordinates of start point and destination. 

As before it generated another JSON request for the tool and our smart map returns a detailed route plan including bus number, how to get to the bus stop, how many stops before getting off, total duration time. 

This process is repeated untill all the places are connected by the transport solution offered by our map MCP. 



















