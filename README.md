# AI Automation Content Pipeline (n8n)

[cite_start]This repository contains an n8n workflow created for the "AI Automation Specialist" assignment[cite: 1]. The project builds a multi-agent pipeline that automates a complete content creation lifecycle: from research to generation and submission.

## Workflow Overview

The entire pipeline is built in n8n and functions as a series of autonomous "Agents" that pass data to each other.

### Agent 1: Content Research Agent

* [cite_start]**Goal:** To discover trending topics related to "AI Automation"[cite: 25].
* **Nodes Used:**
    * `HTTP Request`: Fetches trending searches from Google Trends.
    * [cite_start]`YouTube`: Searches the YouTube Data API for recent, popular videos in the niche[cite: 32].
    * `Code (JavaScript)`: Cleans and parses the JSON output from both sources to extract a simple list of topic titles.
    * `Merge`: Combines the two topic lists.
    * `Remove Duplicates`: Ensures each topic is unique before processing.

### Agent 2: Prompt Agent

* [cite_start]**Goal:** To generate specific, creative prompts for each trending topic[cite: 37].
* **Nodes Used:**
    * `Code (JavaScript)`: This node iterates over each topic and programmatically constructs two detailed prompts:
        1.  A `blog_prompt` for generating a comprehensive article.
        2.  [cite_start]A `video_prompt` for generating a short, engaging video script[cite: 43].

### Agent 3: Content Creator Agent

* [cite_start]**Goal:** To generate a full blog post and a video file for each topic[cite: 46].
* **Implementation:** This agent is split into two loops to manage API rate limits and prevent server overload errors (like `503 Service Unavailable`).

#### Part A: Blog Generation Loop
1.  **Loop Over Items:** Processes each topic one-by-one (`Batch Size = 1`).
2.  **Google Gemini (Text):** Generates a full blog post using the `blog_prompt`.
3.  **Edit Fields:** This critical node re-merges the original `topic` and `video_prompt` with the Gemini output, as the Gemini node drops input data.
4.  **Wait:** A 2-second pause is added to respect API rate limits.
5.  **Code (Clean-up):** After the loop (`done`), this node cleans the 37 blog posts, extracting the text from the `content.parts[0].text` object.

#### Part B: Video Generation Loop
1.  **Loop Over Items:** A second loop (`Batch Size = 1`) processes the 37 items (which now include the blog content).
2.  [cite_start]**Google Gemini (Video):** Simulates the "Google VEO 3 API" [cite: 55] by using the `Generate a video` operation with the `video_prompt`.
3.  **Edit Fields:** This node merges the final video data (e.g., `file_id` or `url`) with the `topic`, `blog_content`, and `video_prompt`.
4.  **Wait:** A 2-second pause for the video API.

### Agent 4: Content Submission Agent

* [cite_start]**Goal:** To submit all generated content to a Google Sheet for human review[cite: 60].
* **Nodes Used:**
    * `Google Sheets`: Connected to the `done` output of the video loop.
    * **Operation:** `Append row in sheet`.
    * [cite_start]**Mapping:** The node maps all 37 items into new rows, filling the 6 columns as required by the assignment [cite: 66-71]:
        1.  `Topic`
        2.  `Prompt` (the video prompt)
        3.  `Blog Content`
        4.  `Video Link` (mapped from the video generation output field)
        5.  `Timestamp` (dynamically generated using `$now`)
        6.  `Status` (set as a static string: "Pending Review")

## Tech Stack & APIs

* **Orchestration:** n8n
* **Research:** Google Trends (via HTTP Request), YouTube Data API v3
* **Content Generation:** Google Gemini API (for both text and video)
* **Submission:** Google Sheets API

## Key Workflow Fixes & Error Handling

Several measures were implemented to ensure the workflow is robust:

1.  **`503 Service Unavailable` Errors:** Solved by placing both `Google Gemini` nodes inside `Loop Over Items` nodes set to `Batch Size = 1`. This prevents sending 37 simultaneous requests.
2.  **`400 Bad Request` (Invalid Value):** Solved on the text generation node by wrapping the `blog_prompt` expression in `JSON.stringify()`.
3.  **Data Loss after Gemini Node:** Solved by adding an `Edit Fields` node immediately after each Gemini node. This node re-adds the original data (like `topic` and `video_prompt`) to the data stream, ensuring it's available for the final Google Sheet.
4.  **General API Failures:** All API-calling nodes (`HTTP Request`, `YouTube`, `Google Gemini`) have the **`Retry On Fail`** setting enabled.
